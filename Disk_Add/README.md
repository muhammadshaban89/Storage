How to  make a newly added hard drive visible on a Linux system **without rebooting**


---

##  Steps to Detect and Mount a New Hard Drive (No Reboot)

### 🔹 1. **Detect the New Disk**
Run this command to rescan the SCSI bus and detect new devices:

echo "- - -" | sudo tee /sys/class/scsi_host/host*/scan

Then verify the new disk is visible:

sudo fdisk -l


Look for a new device like `/dev/sdb`, `/dev/sdc`, etc.

---

### 🔹 2. **Create a Partition (if needed)**
If the disk is unpartitioned:
 
	sudo fdisk /dev/sdX

Replace `X` with the correct letter. Use `n` to create a new partition, then `w` to write changes.

---

### 🔹 3. **Format the Partition**
If you created `/dev/sdX1`, format it:

sudo mkfs.ext4 /dev/sdX1

You can use other file systems like `xfs`, `btrfs`, or `ntfs` depending on your needs.

---

### 🔹 4. **Create a Mount Point**

sudo mkdir /mnt/newdrive

### 🔹 5. **Mount the Drive**

sudo mount /dev/sdX1 /mnt/newdrive


---

### 🔹 6. **Make It Persistent (Optional)**
To auto-mount on boot, add it to `/etc/fstab`:

sudo blkid /dev/sdX1

Copy the UUID and add a line like this to `/etc/fstab`:

UUID=your-uuid-here /mnt/newdrive ext4 defaults 0 2


after removel ,Tell the Kernel to Forget the Device
After unmounting, run:

	echo 1 | sudo tee /sys/block/sdX/device/delete


This command tells the kernel to remove the device from its list. Replace sdX with the correct device name (e.g., sdb).


👉Follow my LinkdIn Profile: www.linkedin.com/in/muhammad-shaban-45577719a

👉Youtube Channel: http://www.youtube.com/@engrm.shaban5099
