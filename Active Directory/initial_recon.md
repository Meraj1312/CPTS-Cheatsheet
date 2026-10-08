# External Recon & Initial AD Enumeration

> **Goal:** Build a target map quickly, validate scope, identify hosts/services/users, and find the first path to valid domain access.
>
> **Core mindset:** `PASSIVE → VALIDATE → ACTIVE → ENUMERATE → REGROUP → ATTACK`
>
> **Rule:** Never attack infrastructure outside written scope. Treat cloud/shared hosting and legacy/industrial systems as potentially fragile.

---

## 1. The Big Picture

### External Recon — What are we looking for?

| Data | What to collect | Why it matters |
|---|---|---|
| **IP Space** | ASN, netblocks, cloud ranges, public IPs | Defines infrastructure footprint + scope validation |
| **Domain Info** | DNS, subdomains, mail, NS, VPN, web portals | Finds exposed services + domain structure |
| **Schema / Naming** | Email format, usernames, AD naming | Builds candidate username lists |
| **Data Disclosures** | PDF/PPT/DOCX/XLSX, source code, metadata | May expose internal hosts, software, users, secrets |
| **Breach Data** | Emails, passwords, hashes | May provide credentials for exposed/internal auth |

### Internal AD Enumeration — Initial targets

1. **Users** → valid usernames for later authentication attacks
2. **Computers** → DCs, file servers, SQL, web, Exchange, admin workstations
3. **Key Services** → `DNS`, `Kerberos`, `LDAP`, `SMB`, `RPC`, `RDP`
4. **Weak / Legacy Hosts** → old OS, exposed services, weak configurations

---

# 2. Methodology

## Passive First

```text
Scope / client info
      ↓
ASN / IP space
      ↓
DNS / subdomains
      ↓
Company website + documents
      ↓
Social media / job postings
      ↓
GitHub / cloud storage
      ↓
Breach data
      ↓
Initial target + username lists
```

## Then Active

```text
Passive host list
      ↓
ICMP / host discovery
      ↓
Port + service enumeration
      ↓
Identify DC / web / SQL / file servers
      ↓
Enumerate AD users
      ↓
Find credentials / hashes / foothold
      ↓
Credentialed AD enumeration
```

### Important habit

**Save everything immediately:**

```bash
mkdir -p recon/{nmap,dns,web,loot,notes}
```

Useful pattern:

```bash
command | tee recon/<name>.txt
```

For Nmap, prefer:

```bash
-oA recon/nmap/<name>
```

Creates `.nmap`, `.gnmap`, and `.xml` output.

---

# 3. External Recon

## ASN / IP Space

### Useful sources

| Resource | Use |
|---|---|
| `bgp.he.net` | ASN, announced prefixes, reverse DNS |
| `arin.net` | North American IP registration |
| `ripe.net` | European IP registration |
| `iana.org` | Internet number / registry reference |
| `viewdns.info` | Reverse IP, DNS, Whois, history |
| `domaintools.com` | Domain / DNS research |
| `lookup.icann.org` | Domain registration data |

### Questions to answer

- What public IPs belong to the organization?
- What ASN(s) are associated with it?
- Is infrastructure self-hosted or on a provider?
- Are there shared/cloud ranges?
- What hosts appear to be mail/web/VPN/DNS?
- Does the discovered infrastructure actually belong to the authorized target?

> **Scope trap:** An IP can host multiple customers. Never assume that because a provider hosts the IP, every reachable tenant is in scope.

---

# 4. DNS Enumeration

## Basic checks

```bash
nslookup example.com
nslookup ns1.example.com
```

```bash
dig example.com
```

Useful record types:

```bash
dig example.com A
dig example.com AAAA
dig example.com MX
dig example.com NS
dig example.com TXT
dig example.com SOA
```

### What to extract

| Record | Look for |
|---|---|
| `A` / `AAAA` | Web/API/VPN/public hosts |
| `MX` | Mail infrastructure |
| `NS` | Authoritative DNS servers |
| `TXT` | SPF, verification strings, other metadata |
| `SOA` | DNS authority / serial / admin clues |
| PTR | Reverse hostname mapping |

### Validate, don't trust one source

```text
BGP / IP source
        ↓
ViewDNS / Whois
        ↓
DNS query (`dig` / `nslookup`)
        ↓
Compare results
```

---

# 5. Public Data / OSINT

## Company Website

Check:

- Contact pages
- Employee names / emails
- About / careers pages
- News / press releases
- Published documents
- Embedded links
- Developer pages / source references
- Intranet / portal references

### Job postings are useful

Look for mentions of:

```text
Microsoft / Windows Server versions
SharePoint
Exchange
Azure / AWS
VMware
Citrix
SQL Server
VPN products
EDR / AV
SIEM
Network vendors
Backup / DR systems
```

**Why:** technology mentions can reveal the environment's likely stack and version history.

---

# 6. Search / Document Hunting

### Google-style dorks

```text
site:example.com filetype:pdf
site:example.com filetype:docx
site:example.com filetype:xlsx
site:example.com filetype:pptx
site:example.com intext:"@example.com"
site:example.com inurl:login
```

Examples from the module:

```text
filetype:pdf inurl:inlanefreight.com
intext:"@inlanefreight.com" inurl:inlanefreight.com
```

### Inspect every document for

```text
Internal URLs
Intranet names
Hostnames
Usernames / email addresses
Network/share references
Software / hardware names
Metadata
Embedded links
Accidental credentials
```

### Preserve findings

```text
Target
URL / source
Downloaded file
Screenshot
Interesting artifact
Why it matters
```

---

# 7. GitHub / Cloud Secret Hunting

Look for:

- Hardcoded credentials
- API keys
- Tokens
- Connection strings
- Internal hostnames
- `.env` files
- Cloud storage references
- Configuration backups

Useful tool/resource examples:

```text
TruffleHog
GitHub search
Greyhat Warfare
```

> **Important:** Finding a secret does not automatically mean it is safe to use. Verify scope and authorization before authentication or exploitation.

---

# 8. Breach Data

Potential sources:

```text
Have I Been Pwned
DeHashed
Other authorized breach-intelligence sources
```

Possible output:

```text
username
email
password
hash
name
historical database source
```

### What matters operationally

```text
Corporate email
       ↓
Username format
       ↓
Potential AD username
       ↓
Candidate password history
       ↓
Test only against authorized authentication targets
```

Older passwords are often stale, but they can still reveal **password patterns** or reused credentials.

---

# 9. Internal Network: Start Passive

## Wireshark

Capture traffic on the relevant interface.

```bash
sudo -E wireshark
```

### Useful traffic clues

| Protocol | What it can reveal |
|---|---|
| **ARP** | Active IPs on local broadcast domain |
| **mDNS** | Hostnames / `.local` names |
| **LLMNR** | Hostname resolution activity |
| **NBT-NS** | NetBIOS host/name activity |
| **DNS** | Hostnames / domains / infrastructure |
| **SMB** | Hosts, shares, Windows activity |
| **Kerberos** | AD/domain presence |

### Passive limitation

A switched network only gives you what your host can legitimately observe on that network segment/broadcast domain. It is not equivalent to seeing all traffic on the subnet.

---

# 10. tcpdump

No GUI? Use:

```bash
sudo tcpdump -i ens224
```

Save a PCAP:

```bash
sudo tcpdump -i ens224 -w recon.pcap
```

Read later:

```bash
tcpdump -r recon.pcap
```

Then open `recon.pcap` in Wireshark.

### Windows built-in option

```text
pktmon.exe
```

---

# 11. Responder — PASSIVE Mode Only Here

For passive analysis:

```bash
sudo responder -I ens224 -A
```

`-A` = **Analyze mode** → listen/analyze without poisoning.

Useful for spotting:

```text
LLMNR
NBT-NS
mDNS
Hostnames
IP addresses
Name-resolution activity
```

> **Remember:** Responder's poisoning functionality is active attack behavior. Do not use it unless the engagement explicitly permits it.

---

# 12. Host Discovery

## fping

### ICMP sweep a CIDR

```bash
fping -asgq 172.16.5.0/23
```

### Flags

| Flag | Meaning |
|---|---|
| `-a` | Show alive hosts |
| `-s` | Print statistics |
| `-g` | Generate target list from CIDR/range |
| `-q` | Quiet mode; suppress per-host output |

### Output → target list

```bash
fping -asgq 172.16.5.0/23 2>/dev/null | tee recon/live-hosts.txt
```

### Critical caveat

**ICMP is not truth.**

A host may be alive but not reply to ICMP because of:

```text
Firewall
Host policy
Network ACL
Filtering
```

So treat fping as **initial validation**, not complete discovery.

---

# 13. Nmap: Initial Service Enumeration

## Quick scan of discovered hosts

```bash
sudo nmap -v -A -iL hosts.txt -oA recon/nmap/host-enum
```

### Important options

| Option | Purpose |
|---|---|
| `-v` | Verbose output |
| `-A` | Aggressive detection set (OS/service/script/trace functions) |
| `-iL hosts.txt` | Read targets from file |
| `-oA name` | Save in `.nmap`, `.gnmap`, `.xml` |

### Better mental model

```text
Host discovery
      ↓
Open ports
      ↓
Service
      ↓
Version
      ↓
Hostname / domain
      ↓
Potential role
      ↓
Follow-up enumeration
```

---

# 14. AD Port Cheat Sheet

## High-value ports to recognize

| Port | Service | Why you care |
|---:|---|---|
| `53` | DNS | Domain infrastructure / name resolution |
| `88` | Kerberos | Strong indicator of AD / KDC |
| `135` | MSRPC | Windows RPC |
| `139` | NetBIOS-SSN | Legacy Windows networking |
| `389` | LDAP | AD directory queries |
| `445` | SMB | Shares, auth, lateral movement |
| `464` | Kerberos password change | AD/Kerberos |
| `593` | RPC over HTTP | Windows RPC |
| `636` | LDAPS | Secure LDAP |
| `3268` | Global Catalog | Forest-wide directory queries |
| `3269` | Global Catalog SSL | Secure Global Catalog |
| `3389` | RDP | Remote Windows access |
| `5985` | WinRM HTTP | Remote PowerShell |
| `5986` | WinRM HTTPS | Secure WinRM |
| `1433` | MSSQL | Database server |
| `80/443` | HTTP/S | Web apps / management portals |

### Strong DC indicators

```text
53
88
389 / 636
445
3268 / 3269
```

Plus Nmap evidence such as:

```text
Active Directory LDAP
Kerberos
Domain name
FQDN
Global Catalog
```

---

# 15. Read Nmap Like a Pentester

When you see:

```text
88/tcp   open kerberos-sec
389/tcp  open ldap
445/tcp  open microsoft-ds
3268/tcp open ldap
```

Think:

```text
This is probably AD infrastructure.
        ↓
What is the domain name?
        ↓
What is the hostname?
        ↓
Is this the DC?
        ↓
What users / directory information can I enumerate?
```

### Nmap clues worth recording

```text
OS version
Hostname
FQDN
NetBIOS name
Domain name
Forest name
Product versions
SMB signing state
RDP NTLM info
LDAP domain/site
TLS certificate names
SQL version
Web server/version
```

---

# 16. Example: Identifying a Domain Controller

Typical output clues:

```text
Host: ACADEMY-EA-DC01
Domain: INLANEFREIGHT.LOCAL
Kerberos: 88
LDAP: 389 / 636
Global Catalog: 3268 / 3269
SMB: 445
```

### Record immediately

```text
DC IP:
DC hostname:
FQDN:
NetBIOS domain:
DNS domain:
Forest:
```

These values become inputs for later Kerberos / LDAP / AD enumeration.

---

# 17. Spotting Legacy Hosts

Example red flags:

```text
Windows Server 2008 R2
IIS 7.5
SQL Server 2008 R2
Old SMB configuration
Old product versions
```

### Why it matters

Potentially means:

```text
Legacy software
Unpatched vulnerabilities
Weak configurations
Legacy authentication
Unsupported OS
```

Possible historical vulnerabilities may include:

```text
MS08-067
EternalBlue / MS17-010
BlueKeep / CVE-2019-0708
```

> **Do not jump straight to exploit.** Legacy systems may be business-critical. Confirm authorization and exploitation rules first, especially for production/OT/ICS environments.

---

# 18. SMB Signing Quick Read

Nmap example:

```text
Message signing enabled but not required
```

Interpretation:

```text
Enabled + required  → stronger protection against SMB relay
Enabled but NOT required → relay may be worth investigating
```

This is a **configuration clue**, not proof that relay will work end-to-end.

---

# 19. Enumerating AD Users with Kerbrute

## Why Kerbrute?

Kerbrute can test candidate usernames against the Kerberos service to identify valid accounts.

### Basic command

```bash
kerbrute userenum -d INLANEFREIGHT.LOCAL \
  --dc 172.16.5.5 \
  jsmith.txt \
  -o valid_ad_users
```

### Meaning

| Part | Meaning |
|---|---|
| `userenum` | Username enumeration |
| `-d` | AD domain |
| `--dc` | Domain Controller IP/host |
| `jsmith.txt` | Candidate username list |
| `-o` | Save results |

### Good wordlist source

```text
Insidetrust / statistically-likely-usernames
```

### Expected output

```text
[+] VALID USERNAME: user@DOMAIN.LOCAL
```

Then:

```text
valid users
    ↓
clean / deduplicate
    ↓
understand naming pattern
    ↓
prepare authorized authentication testing
```

### Lockout warning

Kerberos username enumeration is not the same thing as password spraying, but **badly chosen authentication testing can still cause account lockouts**. Know the client's lockout policy before moving from discovery into password attacks.

---

## Kerbrute

Tool for enumerating and attacking Active Directory accounts via Kerberos pre-authentication.

- **Repo:** https://github.com/ropnop/kerbrute
- **Releases:** https://github.com/ropnop/kerbrute/releases

### Enumeration

Enumerate valid domain usernames (no lockout risk):

```bash
kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.7.3 users.txt
```

### Password Spray

One password against many users (avoids lockout):

```bash
kerbrute passwordspray -d INLANEFREIGHT.LOCAL --dc 172.16.7.3 users.txt 'Password123'
```

Test multiple passwords one at a time:

```bash
while read -r password; do
    kerbrute passwordspray -d INLANEFREIGHT.LOCAL --dc 172.16.7.3 users.txt "$password"
done < common_passwords.txt
```

### Brute Force

Generate every `user:password` combination:

```bash
awk 'NR==FNR{p[++np]=$0; next}{for(i=1;i<=np;i++) print $0 ":" p[i]}' common_passwords.txt users.txt > combos.txt
```

Run Kerbrute against the combinations:

```bash
kerbrute bruteforce -d INLANEFREIGHT.LOCAL --dc 172.16.7.3 combos.txt
```

### Key Concepts

| Technique | Direction | Lockout Risk |
|-----------|-----------|--------------|
| Password spray | one password → many users | Low |
| Brute force | every user → every password | High |

>  **Warning:** Brute force and spraying can lock out accounts. Always check the domain lockout policy first (`net accounts /domain`) and add delays with `-t` (threads) or `--delay` where supported.

### Useful Flags

| Flag | Description |
|------|-------------|
| `-d` | Target domain |
| `--dc` | Domain controller IP |
| `-t` | Threads (default 10) |
| `-o` | Output file |
| `-v` | Verbose |

### Tips

- Use `userenum` first to build a clean `users.txt` (removes invalid accounts).
- Prefer **password spray** over brute force in real engagements.
- Combine with `--delay` / low thread counts to stay under lockout thresholds.
- Output valid creds to a file for use with `crackmapexec`, `impacket`, etc.

```bash
kerbrute passwordspray -d INLANEFREIGHT.LOCAL --dc 172.16.7.3 users.txt 'Password123' -o valid_creds.txt
```

# 21. Getting a Foothold — What Counts?

Early internal AD progress often comes from obtaining **one of these**:

```text
Valid domain credentials
        OR
NTLM password hash
        OR
Shell as a domain user
        OR
SYSTEM on a domain-joined host
```

### Why SYSTEM matters

On a **domain-joined** Windows host, `NT AUTHORITY\SYSTEM` can interact with AD using the machine account context.

This can open paths to:

```text
AD enumeration
BloodHound / PowerView
Kerberoasting
AS-REP roasting
Credential / token abuse
SMB relay-related testing
ACL enumeration / abuse
```

SYSTEM on one host is therefore a major pivot point even without a named domain-user password.

---

# 22. Common SYSTEM Paths to Remember

High-level categories:

```text
Remote OS exploits
Service abuse
SeImpersonate / token impersonation paths
Local privilege escalation
Local admin → SYSTEM execution
```

Historical examples mentioned in the module:

```text
MS08-067
EternalBlue
BlueKeep
Juicy Potato (legacy Windows scenarios)
```

Do not memorize exploits as the methodology. Memorize the **decision path**:

```text
Host identified
    ↓
OS/version
    ↓
Service/version
    ↓
Known weakness?
    ↓
Exploit allowed by scope?
    ↓
Potential stability impact?
    ↓
Exploit / report / escalate for approval
```

---

# 23. Enumeration Notes Template

Use this per host:

```text
[HOST]
IP:
Hostname:
FQDN:
NetBIOS:
Domain:
OS:
Role:

[OPEN PORTS]
53:
88:
135:
139:
389:
445:
636:
3268:
3269:
3389:
5985:
5986:
80/443:
1433:

[KEY FINDINGS]
- 
- 
- 

[POTENTIAL NEXT STEPS]
- 
- 
- 

[SCOPE / RISK]
- 
```

---

# 24. Findings Log — Minimum Useful Format

```text
Time:
Target:
Source/tool:
Finding:
Evidence:
Why interesting:
Confidence:
Next action:
Scope check:
```

### Example

```text
Target: 172.16.5.5
Finding: Likely Domain Controller
Evidence: 53/88/389/445/3268 + INLANEFREIGHT.LOCAL
Why interesting: Central AD infrastructure
Next action: AD/LDAP/Kerberos enumeration
Scope check: Confirmed in-scope
```

---

# 25. CPTS Exam Memory Triggers

### External Recon

```text
ASN → IP space → DNS → subdomains → website → docs → social → GitHub/cloud → breach data
```

### Internal Enumeration

```text
PASSIVE
ARP / mDNS / LLMNR / NBT-NS / DNS
        ↓
ACTIVE
fping / Nmap
        ↓
ROLE IDENTIFICATION
DC / web / SQL / file / admin host
        ↓
USER ENUMERATION
Kerbrute
        ↓
FOOTHOLD
Credentials / hash / shell / SYSTEM
```

### AD ports

```text
53 DNS
88 Kerberos
389 LDAP
445 SMB
636 LDAPS
3268/3269 Global Catalog
3389 RDP
5985/5986 WinRM
```

### DC mental shortcut

```text
88 + 389 + 445 + 3268
        ≈
"Think Domain Controller"
```

### Legacy host mental shortcut

```text
Old OS + old service
        ↓
Potential vulnerability
        ↓
Check exploitability
        ↓
Check scope + stability
```

---

# 26. One-Pass Workflow

```bash
# 1. Create workspace
mkdir -p recon/{nmap,dns,web,loot,notes}

# 2. Passive DNS validation
dig TARGET A
dig TARGET MX
dig TARGET NS
dig TARGET TXT

# 3. Passive network listening
sudo tcpdump -i <iface> -w recon/network.pcap
sudo responder -I <iface> -A

# 4. Initial live-host discovery
fping -asgq <CIDR> 2>/dev/null | tee recon/live-hosts.txt

# 5. Service enumeration
sudo nmap -v -A -iL hosts.txt -oA recon/nmap/host-enum

# 6. Build candidate username list
#    → website / LinkedIn / documents / naming convention / authorized OSINT

# 7. Kerberos username enumeration
kerbrute userenum -d <DOMAIN> --dc <DC-IP> <users.txt> -o recon/valid_ad_users

# 8. Regroup
#    Identify: DCs, users, legacy hosts, interesting services, credentials, next attack paths
```

---

# 27. Do Not Forget

- **Scope is part of enumeration.** An interesting host is not automatically an authorized target.
- **Passive → active.** Start broad and quiet, then validate and narrow.
- **ICMP misses hosts.** Treat fping as an initial signal.
- **Nmap `-A` is noisy and can invoke scripts/detection you may not want in sensitive environments.** Understand what you are running.
- **Save output.** Future enumeration depends on earlier evidence.
- **AD user enumeration is only the beginning.** The goal is usually a valid authenticated foothold.
- **A SYSTEM shell on a domain-joined host is extremely valuable.**
- **Legacy/OT hosts deserve extra caution.** Availability can matter more than speed of exploitation.
- **Do not assume a version alone proves a vulnerability.** Verify configuration, patch level, and exploit conditions.

---

# 28. Useful Resources from the Module

```text
HTB Academy: Footprinting
HTB Academy: OSINT: Corporate Recon
BGP Toolkit: https://bgp.he.net/
ViewDNS: https://viewdns.info/
TruffleHog: https://github.com/trufflesecurity/trufflehog
Kerbrute: https://github.com/ropnop/kerbrute
Insidetrust usernames: https://github.com/insidetrust/statistically-likely-usernames
```
