Network File System (NFS):
-------------------------

- NFS is a protocol that allows you to share directories and files with others over a network.
- It enables users to access files on remote systems as if they were local files. NFS is built into the Linux kernel, making it a powerful and widely available solution for file sharing on Linux systems.

**Setting Up NFS Server**
------

To set up an NFS server, follow these steps:

**1:Install NFS Server Package:**

	  dnf install nfs-utils

**2:Enable and Start NFS Service:**

	 systemctl enable --now nfs-server
   
**3:Create a Directory to Share:** 

     mkdir -p /media/nfs

**4:Configure NFS Exports:**

- Edit the /etc/exports  or    /etc/exports.d/name.exports       (name.exports is naming format for exports .you can use any desirable name )file to specify the directories to share and the clients allowed to access them:

- Add the following line to share the directory with read-write permissions:
  
  * vi /etc/exports  **OR**
  * vi /etc/exports.d/xyz.exports
```
	 /media/nfs   192.168.1.0/24(rw,sync)
```

**4:Export the Shares:**

	 exportfs -arv

**Note: instead of IP you can mention * for all networks , a single host IP , or a specific Network with prefix.**
----

**Some important Points**

| Option | Description |
|--------|-------------|
| **ro** | Export directory as **read‑only**. Clients cannot write. |
| **rw** | Export directory as **read/write**. Clients can read and write. |
| **sync** | Server replies **only after data is written to disk**. Safer but slower. |
| **async** | Server replies **before data is written to disk**. Faster but less safe. |
| **wdelay** | Server **delays writes** if it expects more write requests (optimizes disk I/O). |
| **no_wdelay** | Writes are **not delayed**. Used when `sync` is set. |
| **root_squash** | Maps client‑side root to **nobody** for security. Prevents root access. |
| **no_root_squash** | Client‑side root keeps **root privileges** on the export. Dangerous. |
| **all_squash** | Maps **all client users** to anonymous user (`nobody`). Useful for public shares. |
| **anonuid=UID** | Sets the **UID** for anonymous users (used with squash options). |
| **anongid=GID** | Sets the **GID** for anonymous users. |
| **secure** | Requires clients to use **ports < 1024**. Default for security. |
| **insecure** | Allows clients to use **ports ≥ 1024**. Needed for some clients (e.g., macOS). |
| **subtree_check** | Checks permissions on parent directories. Default for subdirectory exports. |
| **no_subtree_check** | Skips subtree checks. Faster and recommended for most setups. |
| **fsid=0** | Marks the export as the **NFS root** (used for NFSv4). |
| **fsid=NUM** | Assigns a fixed filesystem ID. Useful for exporting the same FS multiple times. |
| **crossmnt** | Allows NFS clients to access **mounted filesystems** under the export. |
| **nohide** | Makes nested NFS mounts visible under the parent export (NFSv3). |



What is root squashing?
-------------

**Root Squashing:**

- By default, root squashing is enabled. This means when a user with root privileges (UID 0) on the client attempts to access the NFS share, their UID is mapped to a specific user on the server, typically "nobody" (UID 65534)

- **no_root_squash :** will disable default behavior of root squashing and client root user will have root privilegs on nfs_share which is security risk.

- So, `no_root_sqaush` must be avoided expect any specific scenario.
- should not use`no_root_squash` unless you are aware of the consequences.

You can test no_root_squash in  as:

on client as root user:
```
	which vi 
```  
- output=/bin/vi

```
cp /bin/vi /nfs_mount
chmod u+s  vi
```
* Now ssh nfs server as non root user:
```
ssh user@nfsserver_ip
```
run:
```
	/nfs_share/vi  /etc/password
```
- and chnage UUID and GID to 0 and save file.

- Logout  and relogging with same account ,you will see that none-priviliged user is now a root user.

- If you try to do so with `root_squash` , it will not let you save file and make any changes.



Connecting to NFS Server from Client
------------------------------------

To connect to the NFS server from a client machine, follow these steps:

**1:Install NFS Client Package:**


	dnf install nfs-utils

**2:Mount the NFS Share:**

	mount -t nfs4 192.168.1.110:/media/nfs    mount_point

**3:Make the Mount Permanent:**

 Add the following line to the client’s /etc/fstab file: 
 ```
192.168.1.110:/media/nfs /media/share nfs4 defaults,user,exec 0 0
```
**4: To execute the fstab changes:**

	mount -a

Note :
----
**Permissions:**

- Ensure that the shared directory has appropriate permissions for the users accessing it.

**Security:**

 - `NFS` does not provide strong security features. Use additional methods to secure sensitive data.
   
**Firewall:**

- Configure your firewall to allow NFS traffic:
```
	firewall-cmd --add-service=nfs --permanent sudo firewall-cmd --reload
```

👉Follow my LinkdIn Profile: www.linkedin.com/in/muhammad-shaban-45577719a

👉Youtube Channel: http://www.youtube.com/@engrm.shaban5099
