**Step_By_Step_GUIDE To Configure ISCSI Target and Initiator on RHEL07**
----------------------------------------------------------------------


**1: set hostname**
  ```bash
  hostnamectl set-hostname iscsihost.example.local
  ```

**2: Install packages.**
- 
  * On Target:
  * 
```bash
yum install targetcli
```
  * On Initiator:
```bash
install iscsi-initiator-utils
```
**3: Enter targetcli shell.**
```bash
targetcli
ls
```
- It will show:
- **BLOCK** = Block-device based LUN Storage Space like Disk e.g. /dev/sdb.
- **FILEIO** = File based LUN Storage Space like file created with dd command.
- **PSCSI** = (SCSI Pass-through) Physical Disk based LUN Storage Space like CD-ROM.
- **RamDisk**= Ramdisk based LUN Storage Space for higher IO speed

**4: Create a Backstore.**
```bash
/backstores/block create backstore01  /dev/sdb
ls
```

**5:  Create ISCI Target**
```bash
/iscsi create
```
- This will :
  
• 	Creates a new iSCSI target node under `/iscsi`

• 	Assigns it a unique IQN (iSCSI Qualified Name)

• 	Initializes a TPG (Target Portal Group) called  `tpg1`

• 	Prepares the structure where you’ll later:

• 	Add LUNs

• 	Configure ACLs

• 	Set portals (IP + port)

• 	Enable authentication


- *A TPG is a container inside an iSCSI target that defines how initiators connect to the target.*

**6: Exposes a real disk (block device) to iSCSI clients by attaching it as a LUN under the target.**  
```bash
/iscsi/iqn.2003-01.org.linux-iscsi.server.x8664:sn.89b50c23d34a/tpg1/luns create /backstores/block/backstore

ls
```

**7: Allows a specific iSCSI initiator to access your target.**

- Go to Client & Copy IQN Number with cp /etc/iscsi/initiatorname.iscsi & return to Host**

- Client Terminal:
```bash
cat /etc/iscsi/initiatorname.iscsi
cp /etc/iscsi/initiatorname.iscsi
````
- Host Terminal: where you left before moving to client terminal:
```bash

/iscsi/iqn.2003-01.org.linux-iscsi.server.x8664:sn.89b50c23d34a/tpg1/acls create iqn.1994-05.com.redhat:3e9d906e8

ls

exit
```
**8: If firewalld is enabled, add port 3260/tcp to firewall rules.**
```bash
sudo firewall-cmd --permanent --add-port=3260/tcp
sudo firewall-cmd --reload
```


Client Configuration: (RHEL 7): 
-------------------------------


**1: Set host Name**
```bash
hostnamectl set-hostname initiator.example.local
hostnamectl
```

**2: install packages**
```bash
yum install -y iscsi-initiator-utils
```

**3: Check available Disks**
```bash
lsblk
```
OR
```
fdisk -l
```
**4: Run Following Commands**
```bash
iscsiadm --help
```
**5: Dicover Target:**
 ```bash
iscsiadm -m discovery -t sendtargets -p 192.168.1.104
```

• 	`-m` discovery → use discovery mode

• 	`-t` sendtargets → use the SendTargets discovery method

• 	`-p` 192.168.1.104 → IP of the iSCSI target server

- Discovery does NOT log in. It only lists available targets.

- **6: Login Target:**  
```bash
iscsiadm -m node -T iqn.2003-01.org.linux-iscsi.server.x8664:sn.89b50c23d34a 192.168.1.104 -l
```

• 	-m node → operate on a discovered node

• 	-T <IQN> → specify which target to log in to

• 	192.168.1.104 → target portal IP

• 	-l → login

📌 Result

• 	A session is created

• 	The kernel receives a new SCSI device

• 	You will see a new disk (e.g., )
Check with:
```
lsblk
```

- OR
```
fdisk -l
```
- OR
```
cat /proc/scsi/scsi
```

**TEST WITH **
```bash
mkdir -p /newdev

mkfs.ext4 /dev/sdb

mount /dev/sdb /newdev

cd /newdev

touch test{1..10}.txt

mkdir aa bb cc dd

ls -lh
```
👉Follow my LinkdIn Profile: www.linkedin.com/in/muhammad-shaban-45577719a

👉Youtube Channel: http://www.youtube.com/@engrm.shaban5099
