

**clear, step‑by‑step CHAP setup** for **one‑way CHAP** (initiator authenticates to target). 

- assume:

- **Target IP:** `192.168.10.10`  
- **Target IQN:** `iqn.2026-01.sa.lab:storage.target01`  
- **Initiator IQN:** `iqn.2026-01.sa.lab:client.node01`  
- **CHAP username:** `chapuser`  
- **CHAP password:** `StrongPass123!`  

You can adapt names as you like.

---

### 1. Enable CHAP on the iSCSI target (RHEL 9.5)

#### 1.1 Enter `targetcli`

```bash
sudo targetcli
```

#### 1.2 Go to the target portal group

```bash
cd /iscsi/iqn.2026-01.sa.lab:storage.target01/tpg1
```

#### 1.3 Require authentication on this TPG

```bash
set attribute authentication=1
```

- This tells the target: “Do not allow sessions without CHAP.”

#### 1.4 Configure CHAP for the specific initiator ACL

Move into the ACL for your initiator:

```bash
cd acls/iqn.2026-01.sa.lab:client.node01
```

Set CHAP username and password:

```bash
set auth userid=chapuser
set auth password=StrongPass123!
```

> This means:  
> - Only an initiator that presents `chapuser` / `StrongPass123!` can log in as `iqn.2026-01.sa.lab:client.node01`.

#### 1.5 Save and exit

```bash
cd /
saveconfig
exit
```

---

### 2. Configure CHAP on the iSCSI initiator (RHEL 9.5)

On the **client**:

#### 2.1 Make sure the initiator IQN matches the ACL

Check:

```bash
sudo cat /etc/iscsi/initiatorname.iscsi
```

If needed, edit:

```bash
sudo vi /etc/iscsi/initiatorname.iscsi
# Set:
InitiatorName=iqn.2026-01.sa.lab:client.node01
```

Restart services later if you change it.

#### 2.2 Discover the target (same as before)

```bash
sudo iscsiadm -m discovery -t sendtargets -p 192.168.10.10
```

You should see:

```text
192.168.10.10:3260,1 iqn.2026-01.sa.lab:storage.target01
```

#### 2.3 Set CHAP parameters for this node

Tell the initiator to use CHAP and provide credentials.

```bash
sudo iscsiadm -m node -T iqn.2026-01.sa.lab:storage.target01 -p 192.168.10.10 \
  --op update -n node.session.auth.authmethod -v CHAP
```

Set username:

```bash
sudo iscsiadm -m node -T iqn.2026-01.sa.lab:storage.target01 -p 192.168.10.10 \
  --op update -n node.session.auth.username -v chapuser
```

Set password:

```bash
sudo iscsiadm -m node -T iqn.2026-01.sa.lab:storage.target01 -p 192.168.10.10 \
  --op update -n node.session.auth.password -v StrongPass123!
```

> These must match exactly what you configured in `targetcli` under the ACL.

#### 2.4 Enable auto‑login (optional but recommended)

```bash
sudo iscsiadm -m node -T iqn.2026-01.sa.lab:storage.target01 -p 192.168.10.10 \
  --op update -n node.startup -v automatic
```

#### 2.5 Log in using CHAP

```bash
sudo iscsiadm -m node -T iqn.2026-01.sa.lab:storage.target01 -p 192.168.10.10 --login
```

If CHAP is correct, the session will establish and a new disk (e.g. `/dev/sdb`) will appear.

Check:

```bash
lsblk
```

---

### 3. Verify CHAP is actually being used

On the **initiator**:

```bash
sudo iscsiadm -m session -P 3
```

Look for:

- `Current Portal: 192.168.10.10:3260`
- `AuthMethod: CHAP`
- `Username: chapuser`

On the **target**, you can re‑enter `targetcli` and check:

```bash
sudo targetcli
cd /iscsi/iqn.2026-01.sa.lab:storage.target01/tpg1/acls/iqn.2026-01.sa.lab:client.node01
ls
```

You should see the `auth` settings you configured.

---

### 4. Quick mental model

- **Without CHAP:**  
  Anyone who can reach port 3260 and knows the IQN can try to log in.

- **With CHAP:**  
  The target requires a **username/password** per initiator IQN.  
  Even if someone spoofs the IQN, they still need the CHAP secret.

👉Follow my LinkdIn Profile: www.linkedin.com/in/muhammad-shaban-45577719a

👉Youtube Channel: http://www.youtube.com/@engrm.shaban5099
