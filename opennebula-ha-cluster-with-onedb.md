### 🛰️ OpenNebula 6.8 HA Cluster Setup Guide (3 Frontends, NFS, Ceph, KVM)

This guide provides a **complete, bit-to-bit** manual installation of OpenNebula 6.8 in a **3-node frontend High Availability (HA)** cluster, with native database failover using `onedb`, and shared storage using **NFS** for image datastore and **Ceph** for system (VM disk) datastore.

---

#### ✅ Architecture Overview

| Component           | Configuration                                 |
|---------------------|-----------------------------------------------|
| Frontend Nodes      | 3 OpenNebula nodes: `one-fe1`, `one-fe2`, `one-fe3` |
| DB Layer            | MariaDB Master-Slave Replication              |
| DB Failover         | Handled natively via `onedb cluster`          |
| Image Datastore     | NFS                                           |
| System Datastore    | Ceph RBD                                      |
| Hypervisors         | KVM                                           |
| Optional VIP        | Keepalived for Sunstone VIP access            |

---

#### 🧰 1. System Preparation (All Nodes)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nfs-common ceph-common sshpass curl wget software-properties-common mariadb-server
```

Set hostname and update `/etc/hosts` across all nodes:

```
192.168.100.10 one-fe1
192.168.100.11 one-fe2
192.168.100.12 one-fe3
192.168.100.20 one-hv1
192.168.100.21 one-hv1
192.168.100.30 one-nfs
```

---

#### 🔐 2. SSH Access for oneadmin

Create oneadmin user and set up SSH keys:

```bash
useradd -m oneadmin
passwd oneadmin
su - oneadmin
ssh-keygen
```

Distribute keys:

```bash
ssh-copy-id oneadmin@one-fe2
ssh-copy-id oneadmin@one-fe3
ssh-copy-id oneadmin@one-hv1
ssh-copy-id oneadmin@one-hv2
```

Ensure consistent UID/GID across all nodes.

---

#### 🐘 3. MariaDB Replication Setup (Master-Slave)

##### On All Frontends

Edit `/etc/mysql/mariadb.conf.d/50-server.cnf`:

```ini
[mysqld]
bind-address = 0.0.0.0
server-id = <1|2|3>
log_bin = /var/log/mysql/mysql-bin.log
```

Restart:

```bash
systemctl restart mariadb
```

##### On Master (e.g., one-fe1)

```sql
CREATE USER 'replicator'@'%' IDENTIFIED BY 'secret';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
FLUSH PRIVILEGES;
```

Dump the DB:

```bash
mysqldump -u root -p --all-databases --master-data > dump.sql
```

Transfer to slaves and import:

```bash
mysql -u root -p < dump.sql
```

Configure replication:

```sql
CHANGE MASTER TO
  MASTER_HOST='one-fe1',
  MASTER_USER='replicator',
  MASTER_PASSWORD='secret',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=xxx;
START SLAVE;
```

---

#### ☁️ 4. OpenNebula Frontend Installation

##### Add Repo and Install

```bash
wget -q -O- https://downloads.opennebula.io/repo/repo.key | sudo gpg --dearmor -o /usr/share/keyrings/opennebula.gpg
echo "deb [signed-by=/usr/share/keyrings/opennebula.gpg] https://downloads.opennebula.io/repo/6.8/Ubuntu/24.04 stable opennebula" | sudo tee /etc/apt/sources.list.d/opennebula.list
sudo apt update
```

Install:

```bash
sudo apt install opennebula opennebula-sunstone opennebula-gate opennebula-flow opennebula-provision opennebula-fireedge -y
```

Edit `/etc/one/oned.conf`:

```ini
DB = [ BACKEND = "mysql",
       SERVER  = "localhost",
       PORT    = 3306,
       USER    = "oneadmin",
       PASSWD  = "your_password",
       DB_NAME = "opennebula" ]
```

Create `/etc/one/one_auth`:

```bash
oneadmin:your_password
```

---

#### 🧠 5. `onedb` HA Configuration

##### On one-fe1 (Master):

```bash
onedb bootstrap -u oneadmin -p your_password
```

##### On one-fe2 and one-fe3:

```bash
onedb change-body -u oneadmin -p your_password --local
```

Enable cluster:

```bash
onedb cluster enable -u oneadmin -p your_password
```

Add nodes:

```bash
onedb cluster add-node one-fe2
onedb cluster add-node one-fe3
```

Start services only on master:

```bash
systemctl enable opennebula opennebula-sunstone
systemctl start opennebula opennebula-sunstone
```

---

#### 🗃️ 6. NFS Image Datastore

##### On NFS Server:

```bash
mkdir -p /var/lib/one/datastores
chown oneadmin:oneadmin /var/lib/one/datastores
echo "/var/lib/one/datastores *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
exportfs -a
```

##### On Frontends and Hypervisors:

```bash
mount one-nfs:/var/lib/one/datastores /var/lib/one/datastores
```

---

### 🧱 OpenNebula and Ceph Integration Guide (Frontend + Hypervisor)

This document outlines **step-by-step instructions** for integrating an existing **Ceph cluster** with **OpenNebula** (frontends and hypervisors) to use **RBD-backed system datastores**.

---

#### 🧠 Assumptions

You already have a working **Ceph cluster** with:

- ✅ One or more **Monitor nodes (MONs)**
- ✅ At least one **OSD node**
- ✅ A pool named `one` (or another you intend to use)
- ✅ Your OpenNebula **frontend and hypervisor nodes are Ceph clients**, not part of the Ceph cluster itself

---

#### 🔍 Step 1: Find Ceph Monitor IPs

On any **Ceph MON node**, run:

```bash
ceph mon dump
```

Example output:

```plaintext
epoch 1
fsid 1fbe4d22-4f63-11ed-82d1-d0abd585bfa3
monmap e1: 3 mons at {
    mon1=192.168.100.41:6789/0
    mon2=192.168.100.42:6789/0
    mon3=192.168.100.43:6789/0
}
```

> These are your monitor IPs. You'll use them to configure `/etc/ceph/ceph.conf`.

---

#### 📁 Step 2: Create Ceph Config File

On **every OpenNebula frontend and hypervisor**, create `/etc/ceph/ceph.conf`:

```ini
[global]
fsid = 1fbe4d22-4f63-11ed-82d1-d0abd585bfa3
mon_host = 192.168.100.41,192.168.100.42,192.168.100.43
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
```

> Replace `fsid` and `mon_host` with values from your Ceph MON node.

---

#### 🔑 Step 3: Create and Distribute Ceph Keyring

##### ✅ Run this **ONLY on a Ceph MON node**:

```bash
ceph auth get-or-create client.oneadmin mon 'allow r' osd 'allow rwx pool=one' -o /etc/ceph/ceph.client.oneadmin.keyring
```

This creates a keyring file for OpenNebula access to the Ceph pool `one`.

##### 📤 Copy the keyring to each OpenNebula node:

```bash
scp /etc/ceph/ceph.client.oneadmin.keyring one-fe1:/etc/ceph/
scp /etc/ceph/ceph.client.oneadmin.keyring one-fe2:/etc/ceph/
scp /etc/ceph/ceph.client.oneadmin.keyring one-fe3:/etc/ceph/
scp /etc/ceph/ceph.client.oneadmin.keyring one-hv1:/etc/ceph/
scp /etc/ceph/ceph.client.oneadmin.keyring one-hv2:/etc/ceph/
```

---

#### 🧩 Step 4: Install ceph-common on Clients

On each frontend and hypervisor node:

```bash
sudo apt install ceph-common -y
```

This provides tools like `rbd`, `ceph`, and libraries to communicate with the Ceph cluster.

---

#### 🔍 Step 5: Test Ceph Access

On any OpenNebula node (frontend or hypervisor):

```bash
rbd -p one ls
```

Expected outcome:
- If empty: just no images yet (✅ success)
- If error: check `ceph.conf`, keyring, or network

---

#### 🧱 Step 6: Create Ceph Datastore in OpenNebula

On the **active OpenNebula frontend**, define the datastore:

```bash
cat > ceph_ds.txt <<EOF
NAME   = "ceph-system"
DS_MAD = ceph
TM_MAD = ceph
DISK_TYPE = RBD
POOL = one
CEPH_USER = oneadmin
CEPH_CONF = /etc/ceph/ceph.conf
EOF

onedatastore create ceph_ds.txt
```

Verify:

```bash
onedatastore list
```

You should now see your `ceph-system` datastore.

---

#### 🧪 Step 7: Use Ceph System Datastore in VM Template

When creating a new VM template in Sunstone or CLI:

- Select the `ceph-system` as the **System Datastore**
- Ensure the **Image Datastore** points to NFS or similar

---

#### ✅ Summary of Key Points

| Task                          | Command / File                                        |
|-------------------------------|--------------------------------------------------------|
| 🔍 Find Ceph MON IPs          | `ceph mon dump`                                       |
| ⚙️ Configure client access     | `/etc/ceph/ceph.conf`                                 |
| 🔑 Create Ceph keyring        | `ceph auth get-or-create ...` (on MON node)           |
| 📤 Distribute keyring         | `scp ... one-feX:/etc/ceph/`                          |
| 📦 Install client tools       | `apt install ceph-common`                             |
| 🧪 Test connectivity          | `rbd -p one ls`                                       |
| 🛠️ Create datastore in ONE    | `onedatastore create ceph_ds.txt`                     |
| 📎 Use in VM template         | Select `ceph-system` for System Datastore             |

---

This guide ensures a clean and correct integration of **Ceph with OpenNebula** using **RBD-backed System Datastore**, across all frontend and hypervisor nodes.


---

#### 🖥️ 8. KVM Hypervisor Setup

```bash
apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils
systemctl enable --now libvirtd
```

Add hypervisor to OpenNebula:

```bash
onehost create one-hv1 -i kvm -v kvm
```

---

#### 🔁 9. Failover & Testing

##### Check Cluster State

```bash
onedb cluster show
```

##### Failover to another node (e.g. one-fe2)

```bash
onedb cluster set-master one-fe2
```

This will:
- Promote one-fe2 DB to master
- Stop/start `oned` service safely
- Ensure OpenNebula points to the new DB master

Check again:

```bash
onedb cluster show
```

##### Test Sunstone

- Browse to `http://<frontend-IP>:9869`
- Verify login, image upload, template creation, VM instantiation

---


You now have a fully functioning **OpenNebula HA cluster** using native tools (`onedb cluster`), with:
- Automatic DB failover
- Centralized image storage via NFS
- RBD-backed VM disk storage via Ceph
- Modular hypervisor addition and full Sunstone functionality

---

#### 💡 Optional Enhancements
- Automate failover logic using watchdog or monitoring triggers
- Add backups and monitoring via Prometheus + Grafana
- Use **Keepalived** for VIP across frontend nodes (for floating Sunstone IP)

### 🛡️ Keepalived Configuration for Database Virtual IP (VIP)

This section explains how to configure **Keepalived** on your OpenNebula frontend nodes to provide a **Virtual IP (VIP)** for MariaDB access. This ensures that the OpenNebula `onedb` connection string always targets a floating IP, managed by Keepalived failover.

---

#### ⚙️ 4. Configure Keepalived for DB VIP

##### 4.1 On FE1 (Master)

###### Install Keepalived:

```bash
sudo apt install keepalived -y
```

###### Configure `/etc/keepalived/keepalived.conf`:

```ini
vrrp_instance VI_1 {
    state MASTER
    interface ens33  # Replace with your actual NIC (e.g., eth0, ens160)
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.100.100  # The shared DB VIP
    }
}
```

> 📝 Make sure the `interface` matches your real network interface name.
> You can find it using `ip addr`.

---

##### 4.2 On FE2 and FE3

- Use the **same config** as FE1
- Change the `state` to `BACKUP`
- Set `priority` to a lower value (e.g., 90, 80)

```ini
state BACKUP
priority 90  # or lower than master
```

---

##### 4.3 Enable and Start Keepalived

On all frontend nodes:

```bash
sudo systemctl enable --now keepalived
```

---

#### 🧪 Testing the VIP

##### From any node on the same network:

```bash
ping 192.168.100.100
```

- Should respond from the current master node
- When master goes down, the VIP will migrate to the next highest priority node

---

#### 📘 Notes

- You can monitor Keepalived logs via `journalctl -u keepalived`
- This VIP can be used in `/etc/one/oned.conf` as the MariaDB `SERVER` value

Example:

```yaml
DB = [ BACKEND = "mysql",
       SERVER  = "192.168.100.100",
       PORT    = 3306,
       USER    = "oneadmin",
       PASSWD  = "your_password",
       DB_NAME = "opennebula" ]
```

---







