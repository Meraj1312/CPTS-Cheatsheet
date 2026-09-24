# Credential Hunting in Windows

> **Core idea:** Understand the machine's role → search likely credential locations → use text searches and application-aware tools → validate anything recovered.

---

# 1. Credential Hunting Mental Model

```text id="mqqc6d"
Compromised Windows machine
        ↓
Who uses this machine?
        ↓
What do they do?
        ↓
What applications do they use?
        ↓
Where would those applications/users store credentials?
        ↓
Search
        ↓
Recover credential material
        ↓
Validate / reuse
```

---

# 2. Search Terms

Start with:

```text id="7s2t1h"
password
passwd
pwd
passphrase
credential
credentials
creds
username
user
login
key
passkey
configuration
dbpassword
dbcredential
```

Also useful:

```text id="0z2m2o"
secret
token
api
apikey
api_key
connectionstring
client_secret
private_key
```

---

# 3. Search by File Content — `findstr`

Module example:

```cmd id="muou6j"
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

Important flags:

```text id="f7zz1v"
/S → recurse subdirectories
/I → case-insensitive
/M → print filename only
/C:"..." → exact search string
```

Mental model:

```text id="91a4mk"
findstr
→ Windows equivalent mindset to grep
```

---

# 4. Search Multiple Credential Terms

Examples:

```cmd id="d5qsw0"
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.ps1 *.yml
```

```cmd id="3ce8s0"
findstr /SIM /C:"credential" *.txt *.ini *.cfg *.config *.xml *.ps1 *.yml
```

```cmd id="us7n5l"
findstr /SIM /C:"secret" *.txt *.ini *.cfg *.config *.xml *.ps1 *.yml
```

Search from a specific directory to keep the output manageable:

```cmd id="7z33r0"
cd C:\Users
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.ps1 *.yml
```

---

# 5. PowerShell Content Search

Useful alternative:

```powershell id="btr4lh"
Get-ChildItem C:\Users -Recurse -File -ErrorAction SilentlyContinue |
Select-String -Pattern 'password','passwd','pwd','credential','secret','apikey' -SimpleMatch -ErrorAction SilentlyContinue
```

Mental model:

```text id="87fl8t"
Get-ChildItem
→ find files

Select-String
→ search contents
```

---

# 6. Search by Filename

Look for interesting filenames:

```powershell id="2mfcrg"
Get-ChildItem C:\Users -Recurse -Force -ErrorAction SilentlyContinue |
Where-Object {
    $_.Name -match 'pass|cred|secret|key|backup'
}
```

Potential names:

```text id="l8hgge"
passwords.txt
pass.txt
credentials.txt
passwords.xlsx
passwords.docx
unattend.xml
web.config
*.kdbx
```

---

# 7. Windows Search

GUI:

```text id="c6nku1"
Windows Search
   ↓
credential-related keyword
   ↓
files/settings/applications
```

Useful first search:

```text id="yyrayv"
password
```

Then:

```text id="xf6j76"
credential
secret
key
login
```

---

# 8. LaZagne

Application-aware credential recovery tool.

Run:

```cmd id="4e6v6k"
LaZagne.exe all
```

More verbose output:

```cmd id="nhi0it"
LaZagne.exe all -vv
```

Check version-specific syntax:

```cmd id="6lbxem"
LaZagne.exe -h
```

---

# 9. LaZagne Modules

| Module     | General target                                       |
| ---------- | ---------------------------------------------------- |
| `browsers` | Browser credentials                                  |
| `chats`    | Chat applications                                    |
| `mails`    | Mail clients                                         |
| `memory`   | Memory-related sources / supported credential stores |
| `sysadmin` | Administrative software such as WinSCP/OpenVPN       |
| `windows`  | Windows credential sources                           |
| `wifi`     | Saved Wi-Fi credentials                              |

---

# 10. Browser Credentials

Potential targets:

```text id="v8om9y"
Chrome / Chromium
Firefox
Edge
Opera
```

Important:

```text id="5at2tw"
Modern browsers
→ stored credentials are encrypted/protected
```

Therefore:

```text id="qu4m4f"
Browser profile
≠
plaintext password file
```

Application-aware tools may be able to recover them depending on access/configuration.

---

# 11. Important Admin-Workstation Targets

If the machine belongs to an IT administrator, think about:

```text id="hj59v9"
WinSCP
PuTTY
OpenVPN
RDP
SSH
KeePass
browser credentials
PowerShell scripts
batch files
configuration files
database clients
remote-management tools
```

Mental model:

```text id="b4y5p3"
Machine role
   ↓
Expected applications
   ↓
Expected credentials
```

---

# 12. High-Value Credential Locations

Keep these in mind:

```text id="fvut4k"
SYSVOL
IT shares
scripts
web.config
unattend.xml
AD user descriptions
AD computer descriptions
KeePass databases
user documents
network shares
SharePoint
```

Potentially interesting files:

```text id="tu3g0m"
passwords.txt
pass.txt
passwords.docx
passwords.xlsx
credentials.txt
config files
deployment scripts
backup scripts
```

---

# 13. SYSVOL

Potential targets:

```text id="g7zqbi"
Group Policy files
deployment scripts
legacy GPP password artifacts
```

Historical artifact to recognize:

```text id="30pk3g"
cpassword
```

It is associated with legacy Group Policy Preferences password storage and should not be considered a secure modern password-storage mechanism.

---

# 14. `web.config`

On Windows web/application systems:

```text id="ux8b1h"
web.config
```

may contain:

```text id="rj1rdx"
database connection strings
application usernames
application passwords
service configuration
```

Search:

```cmd id="r7oaag"
findstr /SIM /C:"password" web.config
```

---

# 15. `unattend.xml`

Windows deployment configurations may contain sensitive setup/authentication information.

Search:

```powershell id="73t7m5"
Get-ChildItem C:\ -Recurse -Filter unattend.xml -ErrorAction SilentlyContinue
```

Then inspect the relevant file carefully.

---

# 16. KeePass

Look for:

```text id="j1o9it"
*.kdbx
```

Example:

```powershell id="q9l09d"
Get-ChildItem C:\Users -Recurse -Filter *.kdbx -ErrorAction SilentlyContinue
```

Important:

```text id="pd4du4"
KDBX found
≠
credentials recovered
```

You still need the KeePass master password/key material.

This connects to:

```text id="ihr4n8"
Protected-file cracking
```

---

# 17. Credential Manager

Enumerate:

```cmd id="l4gq5g"
cmdkey /list
```

This can reveal:

```text id="4v9oy1"
saved credential targets
credential types
associated users
persistence
```

Connects to the previous module:

```text id="z9mdkr"
cmdkey
→ enumerate

runas /savecred
→ reuse
```

---

# 18. LSASS

Credential hunting can overlap with:

```text id="8an0dj"
LSASS
```

Possible workflow:

```text id="a8h7z8"
LSASS dump
   ↓
Pypykatz
   ↓
NT hash / Kerberos / DPAPI / other material
```

This is credential dumping rather than generic file hunting, but both belong in the same post-exploitation process.

---

# 19. Credential Hunting vs Credential Dumping

```text id="4o1e2e"
Credential hunting
→ search broadly for credentials

Credential dumping
→ target known credential stores/processes
```

Examples:

```text id="ly71i8"
Hunting:
findstr
PowerShell
Windows Search
file searches
LaZagne

Dumping:
LSASS
SAM
NTDS.dit
Credential Manager
```

---

# 20. Credential Recovery Hierarchy

Think:

```text id="0dugr0"
Known plaintext credential
        ↓
Usable hash / key / ticket
        ↓
Targeted custom wordlist
        ↓
Rules
        ↓
Generic wordlist
        ↓
Broad brute force
```

Use information before guessing.

---

# 21. Admin Workstation Workflow

```text id="mge0wz"
RDP / CLI access
      ↓
whoami
      ↓
Identify user
      ↓
Identify machine role
      ↓
Enumerate installed applications
      ↓
Search files/configs/scripts
      ↓
cmdkey /list
      ↓
application-aware credential tools
      ↓
browser / VPN / SSH / WinSCP / KeePass
      ↓
Validate recovered credentials
      ↓
Further authorized access
```

---

# 22. Must-Remember Commands

### Identify user

```cmd id="rc31w2"
whoami
```

### Search credentials with findstr

```cmd id="1vbnm5"
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.ps1 *.yml
```

### Search PowerShell

```powershell id="6o8d1s"
Get-ChildItem C:\Users -Recurse -File -ErrorAction SilentlyContinue |
Select-String -Pattern 'password','passwd','pwd','credential','secret' -SimpleMatch -ErrorAction SilentlyContinue
```

### Filename hunting

```powershell id="h778s8"
Get-ChildItem C:\Users -Recurse -Force -ErrorAction SilentlyContinue |
Where-Object { $_.Name -match 'pass|cred|secret|key' }
```

### Credential Manager

```cmd id="wspjhv"
cmdkey /list
```

### LaZagne

```cmd id="jguy2x"
LaZagne.exe all
```

Verbose:

```cmd id="62cuq6"
LaZagne.exe all -vv
```

Help:

```cmd id="9oc9je"
LaZagne.exe -h
```

### Find KeePass databases

```powershell id="x5wev5"
Get-ChildItem C:\Users -Recurse -Filter *.kdbx -ErrorAction SilentlyContinue
```

### Find `unattend.xml`

```powershell id="mk2wwb"
Get-ChildItem C:\ -Recurse -Filter unattend.xml -ErrorAction SilentlyContinue
```

---

# 23. Memory Block

```text id="j7h5wi"
CREDENTIAL HUNTING

First ask:
"What is this machine used for?"

Then search:

CONTENT
→ password
→ passwd
→ pwd
→ credential
→ secret
→ token
→ key
→ login
→ dbpassword
→ dbcredential
→ apikey

FILES
→ *.txt
→ *.ini
→ *.cfg
→ *.config
→ *.xml
→ *.yml
→ *.ps1
→ *.json

LOCATIONS
→ user documents
→ scripts
→ SYSVOL
→ IT shares
→ web.config
→ unattend.xml
→ KeePass
→ SharePoint
→ network shares

TOOLS
→ findstr
→ PowerShell Select-String
→ LaZagne
→ cmdkey
→ application-specific tools
```

---

# 24. One-Line Mental Model

> **Credential hunting is targeted local reconnaissance: understand the user's role and applications, search where credentials are likely to have been stored, recover what you can, and then validate the discovered credential against the next authorized attack path.**
