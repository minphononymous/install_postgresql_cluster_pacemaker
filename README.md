# install_postgresql_cluster_pacemaker
Procedure to Install PostgreSQL Cluster with Pacemaker (RHEL + Pacemaker + PostgreSQL)


# Lab Architecture
Node 1 (db1): Bare Metal RHEL (Active Node)
Node 2 (db2): Bare Metal RHEL (Passive Node)
Node 3 (storage): VM (iSCSI Target / Shared Storage)
VIP: Floating IP address for client connections.

- db1: 192.168.220.11
- db2: 192.168.220.12
- storage: 192.168.220.13
- VIP: 192.168.220.50

# Procedure
> ## Prerequisites
1. Host Resolution (On all nodes) Edit /etc/hosts.
2. Disable SELinux/Firewall (For Lab Only) To minimize friction during learning, stop the firewall. In production, you would whitelist ports 5432, 2224, 3121, 5403, and 9929.

```bash
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config
```
> ## Step 1: Configure Shared Storage

****Node - Storage VM***

Install target CLI:


```bash
dnf install targetcli -y
systemctl enable --now target
```
Create a backing store (Simulate a 10GB disk)
```bash
mkdir -p /var/lib/iscsi_disks
truncate -s 10G /var/lib/iscsi_disks/postgres_data.img
```
Configure iSCSI via targetcli:
```bash
targetcli
```
Inside the shell:
```shell
cd backstores/fileio
create disk01 /var/lib/iscsi_disks/postgres_data.img
cd /iscsi
create iqn.2023-10.com.lab:postgres
cd iqn.2023-10.com.lab:postgres/tpg1/luns
create /backstores/fileio/disk01
cd ../acls

# Allow db1 and db2 to connect (using their initiator names)
create iqn.1994-05.com.redhat:db1
create iqn.1994-05.com.redhat:db2
exit
```

> ## Step 2: Connect Storage (The Bare Metal Nodes)

***On both db1 and db2***

Install iSCSI initiator:

```bash
dnf install iscsi-initiator-utils -y
```
Set the Initiator Name

On db1
```bash
echo "InitiatorName=iqn.1994-05.com.redhat:db1" > /etc/iscsi/initiatorname.iscsi
```

On db2
```bash
echo "InitiatorName=iqn.1994-05.com.redhat:db2" > /etc/iscsi/initiatorname.iscsi
```
Restart iscsid and discover storage:
```bash
systemctl restart iscsid
iscsiadm -m discovery -t st -p 192.168.1.13
iscsiadm -m node -T iqn.2023-10.com.lab:postgres -l
```
Verify the disk: 
```bash
lsblk
```
 You should see a new disk (usually sdx)

> ## Step 3: PostgreSQL & Filesystem Setup
****Run on db1 ONLY***

Bash
```bash
mkfs.xfs /dev/sdb
```
****On BOTH db1 and db2***
```bash
dnf install postgresql-server postgresql-contrib -y
```
Note: Do NOT enable the service (systemctl enable). Pacemaker must control the start/stop.

Move Data Directory (On db1 ONLY): We need to populate the shared disk with the initial DB data.

Mount the shared disk temporarily
```bash
mount /dev/sdb /mnt
```
Initialize the DB directly onto the mount
```bash
mkdir /mnt/data
chown postgres:postgres /mnt/data
chmod 700 /mnt/data
su - postgres -c "initdb -D /mnt/data"
```
Unmount so Pacemaker can take over later
```bash
umount /mnt
```
Update Postgres Config (On BOTH db1 and db2): Edit /var/lib/pgsql/data/postgresql.conf (or the default location) implies local storage. Since we are using a custom location, we will tell Pacemaker where the config is, or we can symlink. For this lab, we will configure the systemd service file to point to the new path.

****On BOTH db1 and db2***
```bash
mkdir -p /etc/systemd/system/postgresql.service.d/
echo "[Service]" > /etc/systemd/system/postgresql.service.d/override.conf
echo "Environment=PGDATA=/var/lib/pgsql/shared_data/data" >> /etc/systemd/system/postgresql.service.d/override.conf
systemctl daemon-reload
```
(Note: We will mount the shared disk to /var/lib/pgsql/shared_data in the cluster config).

> ## Step 4: Install and Configure Pacemaker

****On BOTH db1 and db2***

```bash
dnf install pcs pacemaker corosync fence-agents-all -y
systemctl enable --now pcsd
passwd hacluster
```

On db1 ONLY (Cluster Setup)

Authenticate Nodes:
```bash
pcs host auth db1 db2 -u hacluster -p RedHat123
```

Create and Start Cluster:

Bash
```bash
pcs cluster setup pgsql_cluster db1 db2
pcs cluster start --all
pcs cluster enable --all
```
Disable STONITH (LAB ONLY WARNING):

Critical: In production with shared storage, you MUST have STONITH (fencing) enabled (using IPMI/iLO on your HPE servers) to prevent data corruption. For this lab, we will disable it to save configuration time.

```bash
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
```

> ## Step 5: Define Cluster Resources
We need to tell Pacemaker to manage 3 things in a specific order:
1. Filesystem: Mount the iSCSI disk.
2. Database: Start PostgreSQL.
3. IP: Add the Floating IP.

****On db1***
1. Create Filesystem Resource:

Bash
```bash
pcs resource create pg_fs Filesystem \
    device="/dev/sdb" \
    directory="/var/lib/pgsql/shared_data" \
    fstype="xfs" \
    --group pg_group
```
(Note: We created a group named pg_group. Resources in a group start sequentially on the same node).

Create PostgreSQL Resource:

```bash
pcs resource create pg_service systemd:postgresql \
    --group pg_group
```
Create Virtual IP (VIP) Resource:

```bash
pcs resource create pg_vip IPaddr2 \
    ip=192.168.220.50 \
    cidr_netmask=24 \
    --group pg_group
```
> ## Step 6: Validation & Testing
>>Check Status

```bash
pcs status
```
You should see all resources (fs, service, vip) started on one node (e.g., db1).

Verify Connectivity: From your specific laptop or the storage VM, try to connect to the VIP:

```bash
psql -h 192.168.220.50 -U postgres
```
>>**Simulate Failover: We will crash the active node (db1) to see if db2 takes over.**

****On db1***


```bash
echo c > /proc/sysrq-trigger  'OR simply: reboot'
```

****Watch db2***
```bash
watch -n 1 pcs status
```
- db1 goes OFFLINE.
- pg_fs mounts on db2.
- pg_service starts on db2.
- pg_vip moves to db2.

*The database is now back online on the second server.*
