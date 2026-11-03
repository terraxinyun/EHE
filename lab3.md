# Sharing a Folder from a Win10 VM to the Host

| Item | Value |
|---|---|
| **VM** (the machine that shares the folder) | Win10 VM, IP `172.16.108.164` |
| **HOST** (your own PC, the one that connects) | Logged in as `S11032077`, no admin rights |
| Share name | `shared` → `C:\shared` on the VM |
| Share account (exists on the VM) | `shareuser` / `Pass1234` |

> **🖥️ VM** = type it inside the virtual machine
> **💻 HOST** = type it on your own computer

---

## Quick Overview: Where Each Step Runs

| Step | Where | Command prompt type | What it does |
|---|---|---|---|
| 1 | 🖥️ **VM** | cmd **as Administrator** | Create `C:\shared` |
| 2 | 🖥️ **VM** | cmd **as Administrator** | Share the folder |
| 3 | 🖥️ **VM** | cmd **as Administrator** | Create `test.txt` |
| 4 | 🖥️ **VM** | cmd **as Administrator** | Enable SMB-In firewall rules |
| 5 | 🖥️ **VM** | cmd **as Administrator** | Create `shareuser` account |
| 6 | 🖥️ **VM** | cmd **as Administrator** | Give `shareuser` folder permissions |
| 7 | 🖥️ **VM** | cmd **as Administrator** | Verify share, get name and IP |
| 8 | 💻 **HOST** | normal cmd | Clear old connections |
| 9 | 💻 **HOST** | normal cmd | Connect to the share as `shareuser` |
| 10 | 💻 **HOST** | normal cmd | Test access |
| 11 | 💻 **HOST** | normal cmd | (Optional) Map as drive Z: |
| 12 | 💻 **HOST** | normal cmd | Remove the mapped drive |

---

## Part A: 🖥️ On the VM

> Open **Command Prompt as Administrator** inside the VM:
> Start → type `cmd` → right-click **Command Prompt** → **Run as administrator**

### Step 1 (🖥️ VM): Create the folder
```cmd
mkdir C:\shared
```
Creates the folder on the VM. If it says it already exists, ignore it.

### Step 2 (🖥️ VM): Share it on the network
```cmd
net share shared=C:\shared /grant:Everyone,FULL
```
Publishes the VM's `C:\shared` on the network as `\\172.16.108.164\shared`. `/grant:Everyone,FULL` sets the **share permissions** so any authenticated user can read and write. If it says the share already exists, it's already shared.

### Step 3 (🖥️ VM): Put some data in it
```cmd
echo This is test data for the shared folder. > C:\shared\test.txt
```
Creates `test.txt` on the VM, so the host has something to read.

### Step 4 (🖥️ VM): Open the firewall for file sharing
```cmd
netsh advfirewall firewall set rule name="File and Printer Sharing (SMB-In)" new enable=yes
```
Enables the VM's inbound firewall rules for **SMB (TCP port 445)**, the protocol Windows file sharing uses. Without this, the host can't reach the VM's share. It covers the **Domain, Private and Public** profiles.

### Step 5 (🖥️ VM): Create an account the host can log in with
```cmd
net user shareuser Pass1234 /add
```
Creates a local account **on the VM**. Your host user (`S11032077`) doesn't exist on the VM, which causes **"System error 5 – Access is denied"**. The host logs in to the VM as `shareuser` instead.

### Step 6 (🖥️ VM): Give that account folder permissions
```cmd
icacls C:\shared /grant shareuser:(OI)(CI)F
```
The VM checks **two permission layers**:
- **Share permissions**, set in Step 2
- **NTFS folder permissions**, set in this step

Both must allow access. `F` is full control. `(OI)(CI)` applies it to all files and subfolders.

### Step 7 (🖥️ VM): Verify and get the VM's details
```cmd
net share shared
hostname
ipconfig
```
| Command | Purpose |
|---|---|
| `net share shared` | Confirms the share exists on the VM |
| `hostname` | Shows the VM's computer name (needed if Step 9 fails) |
| `ipconfig` | Shows the VM's IP address (should be `172.16.108.164`) |

✅ **VM setup is done. Switch to your host.**

---

## Part B: 💻 On the HOST

> Open a **normal Command Prompt** on your own PC (no admin needed):
> Start → type `cmd` → **Command Prompt**

### Step 8 (💻 HOST): Clear any old connections
```cmd
net use * /delete /y
```
Removes cached connections and credentials on your host from earlier failed attempts.

### Step 9 (💻 HOST): Connect using the VM account
```cmd
net use \\172.16.108.164\shared /user:172.16.108.164\shareuser Pass1234
```
Your host logs in to the VM's share as `shareuser`. The `172.16.108.164\` prefix tells Windows it's a **local account on the VM**, not on your host.

Expected output: `The command completed successfully.`

> If this fails, use the VM's computer name from Step 7 instead, e.g. `/user:DESKTOP-ABC123\shareuser`

### Step 10 (💻 HOST): Test access
```cmd
net view \\172.16.108.164
dir \\172.16.108.164\shared
type \\172.16.108.164\shared\test.txt
```
| Command | Purpose |
|---|---|
| `net view` | Lists the shares on the VM |
| `dir` | Lists the files in the VM's shared folder |
| `type` | Prints the contents of `test.txt` from the VM |

### Step 11 (💻 HOST, optional): Map it as a drive
```cmd
net use Z: \\172.16.108.164\shared /user:172.16.108.164\shareuser Pass1234
```
The VM's share appears as drive **Z:** in File Explorer on your host.

### Step 12 (💻 HOST): Remove the mapped drive / disconnect
```cmd
net use Z: /delete
```
Removes the **Z:** drive from your host. If files are open and it asks to confirm, add `/y`:
```cmd
net use Z: /delete /y
```
To remove **all** mapped drives and connections from your host at once:
```cmd
net use * /delete /y
```
Confirm nothing is left:
```cmd
net use
```
Expected output: `There are no entries in the list.`

> Removing the mapped drive on the host does **not** delete the share or files on the VM.

---

## How It Fits Together

```
 💻 HOST (S11032077)                         🖥️ VM (172.16.108.164)
 ───────────────────                         ──────────────────────
 net use / dir / type  ──SMB port 445──►  Firewall SMB-In allowed  (Step 4)
   (Steps 8–12)                                    │
                                          Login as shareuser       (Step 5)
                                                   │
                                   Share permission: Everyone FULL  (Step 2)
                                                   │
                                 NTFS permission: shareuser FULL    (Step 6)
                                                   │
                                         C:\shared\test.txt ✔       (Steps 1, 3)
```

---

## Troubleshooting

| Error (seen on 💻 HOST) | Meaning | Fix |
|---|---|---|
| **System error 53** | Network path not found | 🖥️ VM firewall or network blocking. Check Step 4, and ping the VM from the host |
| **System error 5** | Access is denied | Login or permissions wrong. Check 🖥️ Steps 5 and 6, and 💻 Step 9 |
| **System error 86** | Incorrect password | Re-check the password used in 💻 Step 9 |
| **System error 1219** | Multiple connections to same server | Run 💻 Step 8 again |
| **Guest access blocked** | Host blocks unauthenticated guest logons | Use the account method (🖥️ Step 5 and 💻 Step 9), not guest |
