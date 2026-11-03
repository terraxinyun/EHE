# Sharing a Folder from a Win10 VM to the Host

**VM IP:** `172.16.108.164`
**Share name:** `shared` → `C:\shared`
**Share account:** `shareuser` / `Pass1234`

---

## Part A: On the VM (Command Prompt as Administrator)

### Step 1: Create the folder
```cmd
mkdir C:\shared
```
Creates the folder. If it says it already exists, ignore it.

### Step 2: Share it on the network
```cmd
net share shared=C:\shared /grant:Everyone,FULL
```
Publishes `C:\shared` as `\\<VM-IP>\shared`. `/grant:Everyone,FULL` sets the **share permissions** so any authenticated user can read and write. If it says the share already exists, it's already shared.

### Step 3: Put some data in it
```cmd
echo This is test data for the shared folder. > C:\shared\test.txt
```
Creates `test.txt` with one line of text, so there's something to read from the host.

### Step 4: Open the firewall for file sharing
```cmd
netsh advfirewall firewall set rule name="File and Printer Sharing (SMB-In)" new enable=yes
```
Enables the built-in inbound rules for **SMB (TCP port 445)**, the protocol Windows file sharing uses. Without this, the host can't reach the share. It covers the **Domain, Private and Public** profiles.

### Step 5: Create an account the host can log in with
```cmd
net user shareuser Pass1234 /add
```
Creates a local VM account. The host user (e.g. `S11032077`) doesn't exist on the VM, which causes **"System error 5 – Access is denied"**. The host logs in as this user instead.

### Step 6: Give that account folder permissions
```cmd
icacls C:\shared /grant shareuser:(OI)(CI)F
```
Windows checks **two permission layers**:
- **Share permissions**, set in Step 2
- **NTFS folder permissions**, set in this step

Both must allow access. `F` is full control. `(OI)(CI)` applies it to all files and subfolders.

### Step 7: Verify and get the VM's details
```cmd
net share shared
hostname
ipconfig
```
| Command | Purpose |
|---|---|
| `net share shared` | Confirms the share exists |
| `hostname` | Shows the VM's computer name |
| `ipconfig` | Shows the VM's IP address |

---

## Part B: On the Host (normal Command Prompt, no admin needed)

### Step 8: Clear any old connections
```cmd
net use * /delete /y
```
Removes cached connections and credentials from earlier failed attempts.

### Step 9: Connect using the VM account
```cmd
net use \\172.16.108.164\shared /user:172.16.108.164\shareuser Pass1234
```
Logs in to the share as `shareuser`. The `172.16.108.164\` prefix tells Windows it's a **local account on the VM**.

Expected output: `The command completed successfully.`

> If this fails, use the VM's computer name from Step 7 instead, e.g. `/user:DESKTOP-ABC123\shareuser`

### Step 10: Test access
```cmd
net view \\172.16.108.164
dir \\172.16.108.164\shared
type \\172.16.108.164\shared\test.txt
```
| Command | Purpose |
|---|---|
| `net view` | Lists the VM's shares |
| `dir` | Lists files in the share |
| `type` | Prints the contents of `test.txt` |

### Step 11 (optional): Map it as a drive
```cmd
net use Z: \\172.16.108.164\shared /user:172.16.108.164\shareuser Pass1234
```
The share appears as drive **Z:** in File Explorer.

### Step 12: Remove the mapped drive / disconnect
```cmd
net use Z: /delete
```
Removes the **Z:** drive mapping. If files are open and it asks to confirm, add `/y`:
```cmd
net use Z: /delete /y
```
To remove **all** mapped drives and connections to the VM at once:
```cmd
net use * /delete /y
```
Confirm nothing is left:
```cmd
net use
```
Expected output: `There are no entries in the list.`

---

## How It Fits Together

```
Host (S11032077)  ──SMB port 445──►  VM Firewall        (Step 4)
                                         │
                                Login as shareuser     (Step 5)
                                         │
                         Share permission: Everyone FULL (Step 2)
                                         │
                       NTFS permission: shareuser FULL  (Step 6)
                                         │
                               C:\shared\test.txt ✔
```

---

## Troubleshooting

| Error | Meaning | Fix |
|---|---|---|
| **System error 53** | Network path not found | Firewall or network blocking. Check Step 4 and ping the VM |
| **System error 5** | Access is denied | Login or permissions wrong. Check Steps 5, 6, 9 |
| **System error 86** | Incorrect password | Re-check the password |
| **System error 1219** | Multiple connections to same server | Run Step 8 again |
| **Guest access blocked** | Host blocks unauthenticated guest logons | Use the account method (Steps 5 and 9), not guest |
