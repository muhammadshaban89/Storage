
## 1. What is iSCSI?

**iSCSI (Internet Small Computer Systems Interface)** is a protocol that sends SCSI commands over IP networks.  
Instead of directly attaching a disk to a server, you expose block storage over the network, and the client sees it as a local disk.

- **Type:** Block storage over TCP/IP  
- **Use cases:** SAN, virtualization storage, shared LUNs, lab storage, etc.   

---

## 2. iSCSI target vs initiator

- **iSCSI Target:**
  - **What it is:** The storage server that *exports* block devices (LUNs) over the network.
  - **Role:** Owns the physical storage (disk, LVM, file-backed disk) and presents it as iSCSI LUNs.
  - **Think of it as:** “The storage box.”

- **iSCSI Initiator:**
  - **What it is:** The client that *connects* to the target to use its LUNs.
  - **Role:** Discovers targets, logs in, and uses the exposed LUNs as if they were local disks.
  - **Think of it as:** “The server/host consuming the storage.”   

In your lab terms:  
- ESXi/KVM/K8s node → **initiator**  
- Storage VM or bare-metal storage box → **target**

---

## 3. Packages needed on RHEL 9.5

### On the iSCSI target (storage server)

- **Package:** `targetcli`  
  - Provides the `targetcli` shell to configure LIO (Linux I/O Target), the in-kernel iSCSI target framework.   

```bash
sudo dnf install targetcli
```

> On RHEL 9, LIO is the default iSCSI target implementation, managed via `targetcli`.

### On the iSCSI initiator (client/host)

- **Package:** `iscsi-initiator-utils`  
  - Provides `iscsiadm`, `iscsid`, and related tools to discover and log in to iSCSI targets.   

```bash
sudo dnf install iscsi-initiator-utils
```

---

## 4. Step-by-step: configure iSCSI target on RHEL 9.5

Let’s assume:

- **Target (server) IP:** `192.168.10.10`
-  create a **file-backed LUN** 

### 4.1 Install and enable target services

```bash
sudo dnf install targetcli -y
sudo systemctl enable --now target.service
```

- **What this does:**  
  - Installs the management tool.  
  - Starts the LIO target framework and ensures it comes up on boot.

### 4.2 Prepare backing storage ----(Section-1) 

**Note**

*If you want to use a block storage as backing storage follow "section-2" in same document.*

Example: create a 10 GB file to act as a virtual disk.

```bash
sudo mkdir -p /iscsi_disks
sudo dd if=/dev/zero of=/iscsi_disks/disk01.img bs=1G count=10
```

- **Why:**  
  - In production you’d usually use a real block device or LVM LV.  
  - For labs, a file-backed LUN is simple and flexible.

### 4.3 Enter targetcli shell

```bash
sudo targetcli
```

You’ll see a prompt like:

```text
/> 
```

### 4.4 Create a backstore

```bash
/backstores/fileio create disk01 /iscsi_disks/disk01.img
```

- **Meaning:**
  - **`fileio`**: backstore type using a file as backing storage.  
  - **`disk01`**: name of the backstore.  
  - **`/iscsi_disks/disk01.img`**: path to the file.

You can also use `/backstores/block` if you want to expose a real block device (e.g. `/dev/sdb`).

### 4.5 Create an iSCSI target

```bash
/iscsi create
```

This auto-generates an IQN like:

```text
iqn.2026-01.com.example:target01
```

- **IQN (iSCSI Qualified Name):**  
  - Uniquely identifies the target.  
  - You can also specify your own, e.g.:

```bash
/iscsi create iqn.2026-01.sa.lab:storage.target01
```

### 4.6 Create a LUN for that target

```bash
/iscsi/iqn.2026-01.sa.lab:storage.target01/tpg1/luns create /backstores/fileio/disk01
```

- **What this does:**
  - Binds the backstore `disk01` to the target portal group `tpg1` as a LUN.
  - This is what the initiator will see as a disk.

### 4.7 Configure ACLs (initiator access)

You can either:

- Allow all initiators (not recommended for production), or  
- Restrict to specific initiator IQNs (better).

Let’s assume your initiator IQN will be:

```text
iqn.2026-01.sa.lab:client.node01
```

Create ACL:

```bash
/iscsi/iqn.2026-01.sa.lab:storage.target01/tpg1/acls create iqn.2026-01.sa.lab:client.node01
```

- **Why:**  
  - Only this initiator IQN will be allowed to log in and see the LUN.

### 4.8 Configure network portal

By default, `tpg1` usually has a portal on `0.0.0.0:3260`. If not, create it:

```bash
/iscsi/iqn.2026-01.sa.lab:storage.target01/tpg1/portals create 0.0.0.0 3260
```

- **Meaning:**  
  - Listens on all interfaces on port `3260` (standard iSCSI port).

### 4.9 Save configuration and exit

```bash
saveconfig
exit
```

- **Why:**  
  - Persists the configuration so it survives reboot.

### 4.10 Firewall configuration (if firewalld is enabled)

```bash
sudo firewall-cmd --permanent --add-port=3260/tcp
sudo firewall-cmd --reload
```

---

## 5. Step-by-step: configure iSCSI initiator on RHEL 9.5

Assume:

- **Initiator (client) IP:** `192.168.10.20`  
- **Target IP:** `192.168.10.10`  
- **Target IQN:** `iqn.2026-01.sa.lab:storage.target01`

### 5.1 Install initiator utilities

```bash
sudo dnf install iscsi-initiator-utils -y
```

### 5.2 Check or set initiator IQN

File: `/etc/iscsi/initiatorname.iscsi`

```bash
sudo cat /etc/iscsi/initiatorname.iscsi
```

You’ll see something like:

```text
InitiatorName=iqn.2026-01.com.redhat:client
```

If you want to match what you used in the target ACL:

```bash
sudo vi /etc/iscsi/initiatorname.iscsi
# Set:
InitiatorName=iqn.2026-01.sa.lab:client.node01
```

Then restart the service later.

### 5.3 Enable and start iSCSI services

```bash
sudo systemctl enable --now iscsid
sudo systemctl enable --now iscsi
```

- **`iscsid`:** iSCSI daemon handling sessions.  
- **`iscsi`:** manages automatic logins and sessions.   

### 5.4 Discover targets on the server

```bash
sudo iscsiadm -m discovery -t sendtargets -p 192.168.10.10
```

You should see output like:

```text
192.168.10.10:3260,1 iqn.2026-01.sa.lab:storage.target01
```

- **Meaning:**  
  - The initiator queried the target and got a list of available IQNs.

### 5.5 Log in to the target

```bash
sudo iscsiadm -m node -T iqn.2026-01.sa.lab:storage.target01 -p 192.168.10.10 --login
```

- **What happens now:**  
  - The kernel creates a new SCSI device (e.g. `/dev/sdb`).

To make login persistent across reboot:

```bash
sudo iscsiadm -m node -T iqn.2026-01.sa.lab:storage.target01 -p 192.168.10.10 --op update -n node.startup -v automatic
```

### 5.6 Verify the new disk

```bash
lsblk
```

You should see something like:

```text
sda    100G
└─sda1 ...
sdb     10G
```

- **`/dev/sdb`** is your iSCSI LUN.

### 5.7 Partition, format, and mount

Example using a single partition and ext4:

```bash
sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/sdb1
sudo mkdir -p /mnt/iscsi_lun
sudo mount /dev/sdb1 /mnt/iscsi_lun
```

Check:

```bash
df -h /mnt/iscsi_lun
```

### 5.8 Make mount persistent (optional)

Get the UUID:

```bash
blkid /dev/sdb1
```

Add to `/etc/fstab`:

```text
UUID=<the-uuid-here>  /mnt/iscsi_lun  ext4  _netdev  0  0
```

- **`_netdev`:** tells the system this filesystem depends on the network.

---

## 6. Optional: CHAP authentication (high-level)

For production, you’d usually protect the target with CHAP:

- On the **target**: configure CHAP username/password in `tpg1` auth settings.  
- On the **initiator**: set the same credentials using `iscsiadm` node parameters.

--------------------------------------------------------------------------------

### Section-2

- if you want to use **block storage** instead of a **file‑backed LUN**, the setup becomes even cleaner.  
- You simply expose a **real block device** (like `/dev/sdb`) directly through the iSCSI target.
-  **correct, clean, step‑by‑step method** for RHEL 9.5 using **block backstore**.

---

# 1. Prepare a real block device

You do **NOT** create a file with `dd`.  
Instead, you use a real disk such as:

```
/dev/sdb
```

If you want to use an LVM logical volume, that also works:

```
/dev/vg_iscsi/lv01
```

---

# 2. Configure iSCSI Target with Block Backstore

Enter the targetcli shell:

```bash
sudo targetcli
```

### 2.1 Create a block backstore

Example using `/dev/sdb`:

```bash
/backstores/block create block01 /dev/sdb
```

Explanation:
- **block01** → name of the backstore  
- **/dev/sdb** → real block device exported to initiators  

### 2.2 Create the iSCSI target

```bash
/iscsi create
```

This generates an IQN like:

```
iqn.2026-01.sa.lab:storage.target01
```

### 2.3 Create the LUN using the block backstore

```bash
/iscsi/iqn.2026-01.sa.lab:storage.target01/tpg1/luns create /backstores/block/block01
```

### 2.4 Add ACL for your initiator

Replace with your initiator IQN:

```bash
/iscsi/iqn.2026-01.sa.lab:storage.target01/tpg1/acls create iqn.2026-01.sa.lab:client.node01
```

### 2.5 Add network portal (if needed)

```bash
/iscsi/iqn.2026-01.sa.lab:storage.target01/tpg1/portals create 0.0.0.0 3260
```

### 2.6 Save and exit

```bash
saveconfig
exit
```

---

#  3. Firewall (if enabled)

```bash
sudo firewall-cmd --permanent --add-port=3260/tcp
sudo firewall-cmd --reload
```

---

# 4. Initiator (Client) — No change needed

The client sees the block LUN exactly the same way as fileio LUNs.

Steps:

```bash
sudo iscsiadm -m discovery -t sendtargets -p <TARGET_IP>
sudo iscsiadm -m node --login
lsblk
```

You will see something like:

```
sdb   20G   # <-- this is your block LUN
```

Then partition/format/mount as usual.

---

#  Summary: FileIO vs Block Backstore

| Type | Backing | Use Case |
|------|---------|----------|
| **fileio** | File like `/iscsi_disks/disk01.img` | Labs, testing, flexible |
| **block** | Real disk `/dev/sdb` or LVM LV | Production, better performance |



👉Follow my LinkdIn Profile: www.linkedin.com/in/muhammad-shaban-45577719a

👉Youtube Channel: http://www.youtube.com/@engrm.shaban5099



