# Windows Authentication Process — CPTS Cheatsheet

**Core idea:** Windows logon flows through Winlogon → LogonUI/Credential Provider → LSA/LSASS → authentication package → local SAM or domain authentication authority.

---

## 1. Main Flow

```
User
 ↓
Winlogon
 ↓
LogonUI
 ↓
Credential Provider
 ↓
LSA / LSASS
 ↓
Authentication Package
 ↓
SAM (local)
OR
Domain Controller / Active Directory (domain)
 ↓
Accept / Reject
```

---

## 2. Components

| Component | Remember it as |
|---|---|
| `Winlogon.exe` | Interactive logon coordinator |
| `LogonUI.exe` | Login interface |
| Credential Provider | Collects/packages credentials |
| LSA | Windows security authority |
| `LSASS.exe` | LSA subsystem process |
| `Lsasrv.dll` | LSA Server service/security package manager |
| `Msv1_0.dll` | MSV1_0 authentication package |
| `Kerberos.dll` | Kerberos security package |
| `Samsrv.dll` | SAM-related service |
| `Netlogon.dll` | Domain logon / secure-channel functionality |
| `Ntdsa.dll` | Active Directory directory service component on DCs |

---

## 3. Winlogon

Responsible for interactive security-related user interaction:

```
Logon
Password change
Workstation lock
Workstation unlock
```

```
Winlogon
 ↓
starts/coordinates LogonUI
 ↓
passes collected credentials to security subsystem
```

---

## 4. LogonUI

Provides the graphical login experience.

```
Winlogon
 ↓
LogonUI
 ↓
Credential Provider
```

---

## 5. Credential Providers

Modern Windows mechanism for collecting credentials.

```
Password
PIN
Smart Card
Windows Hello
Biometrics
```

```
Credential Provider
→ collects/packages credentials

LSA/authentication packages
→ enforce authentication/security
```

---

## 6. GINA (legacy)

```
GINA       → legacy (XP / Server 2003)
Credential Provider → modern
```

---

## 7. LSA vs LSASS

**LSA** (Local Security Authority) — authentication, local security policy, SID translation, security-related functions.

**LSASS** (Local Security Authority Subsystem Service) — the process: `C:\Windows\System32\lsass.exe`

```
LSA = authority/subsystem
LSASS = process
```

---

## 8. Why LSASS Matters

LSASS can maintain authentication/session credential material in memory:

```
NT hash
LM hash
Kerberos TGT
Kerberos service tickets
other credential forms
```

```
Active Windows session
        ↓
LSASS
        ↓
credential material may exist in memory
```

Therefore: privileged LSASS access = a potential credential extraction target. See `attacking_lsass.md`.

---

## 9. Authentication Packages

| DLL | Role |
|---|---|
| `Msv1_0.dll` | MSV1_0 authentication package — NTLM/local authentication |
| `Kerberos.dll` | Kerberos authentication/security package — important in AD |
| `Lsasrv.dll` | LSA Server — security policy/package management |
| `Netlogon.dll` | Domain logon / secure channel — domain members ↔ DCs |
| `Samsrv.dll` | SAM service — manages local account/security database |
| `Ntdsa.dll` | AD directory service component — loaded on DCs |

---

## 10. Negotiate

Windows uses **Negotiate** to select an authentication protocol:

```
Authentication request
       ↓
Negotiate
       ↓
Kerberos or NTLM
```

---

## 11. SAM

**Security Accounts Manager** — stores local Windows account information/credential material.

```
File:     C:\Windows\System32\config\SAM
Registry: HKLM\SAM
```

```
SAM
 ↓
LOCAL ACCOUNTS
```

See `sam_system_security.md` for the full attack.

---

## 12. Workgroup vs Domain

```
Workgroup:
Computer A → local accounts
Computer B → local accounts
(each system manages its own accounts)

Domain:
                 Domain Controller
                        ↓
                 Active Directory
                  /    |    \
              PC01    PC02   Server
(centrally managed)
```

```
LOCAL ACCOUNT  → SAM
DOMAIN ACCOUNT → Domain Controller → Active Directory
```

---

## 13. NTDS.dit

Active Directory database on a Domain Controller.

```
C:\Windows\NTDS\ntds.dit
```

```
NTDS.dit
 ↓
DOMAIN
```

See `active_directory_ntds.md` for the full attack.

---

## 14. Credential Manager

Windows feature for storing saved credentials (network resources, apps, websites, remote systems). Stored in encrypted user-profile storage.

```
User saves credential
        ↓
Credential Manager
        ↓
encrypted storage
        ↓
application reuses credential
```

See `credential_manager.md` for the full attack.

---

## 15. The Four Credential Locations

```
SAM
→ local account database

NTDS.dit
→ domain account database

LSASS
→ active-session credential material in memory

Credential Manager
→ saved credentials
```

```
SAM     → LOCAL
NTDS    → DOMAIN
LSASS   → LIVE / MEMORY
CredMan → SAVED
```

---

## 16. Attacker Perspective

```
What users exist?          → SAM?
What credentials active?   → LSASS?
What was saved?            → Credential Manager?
Domain joined?              → AD / Kerberos?
Is this a DC?                → NTDS.dit?
```

---

## 17. Credential Material Is Not Always a Password

```
Plaintext password
NT hash
LM hash
Kerberos TGT
Kerberos service ticket
Cached credential
Saved credential
Private key
```

Don't equate `credential = plaintext password`.

---

## 18. Historical Syskey Note

`Syskey` historically added protection around SAM-related data. On modern Windows it's **obsolete/unsupported** — don't treat it as a current feature.

---

## 19. Must-Remember Paths

```
# LSASS
C:\Windows\System32\lsass.exe

# SAM
C:\Windows\System32\config\SAM

# Active Directory database
C:\Windows\NTDS\ntds.dit

# Credential Manager-related user profile area
C:\Users\<Username>\AppData\Local\Microsoft\
```

---

## 20. Memory Block

```
WINDOWS AUTHENTICATION

Winlogon            → coordinates interactive logon
LogonUI              → login interface
Credential Provider  → collects/packages authentication data
LSA                  → security authority
LSASS                → LSA subsystem process / credential material in memory
SAM                  → LOCAL account database
NTDS.dit             → DOMAIN / Active Directory database
Credential Manager   → SAVED credentials
Msv1_0               → MSV1_0 / NTLM-related authentication
Kerberos.dll         → Kerberos
Netlogon.dll         → domain logon / secure channel
Ntdsa.dll            → Active Directory directory service
Lsasrv.dll           → LSA Server
```

---

## 21. One-Line Mental Model

> Windows collects credentials through Winlogon/LogonUI/Credential Providers, authenticates them through LSA/LSASS and authentication packages, and ultimately validates local accounts against SAM or domain accounts through Active Directory.
