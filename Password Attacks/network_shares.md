# Credential Hunting in Network Shares

## Core Concept

Network shares are remote filesystems that may contain:

- Passwords
- Credentials
- Config files
- Scripts
- SSH keys
- API keys
- Tokens
- Backups
- Deployment files
- Sensitive documents

Core workflow:

```
Credentials
    ↓
Enumerate computers
    ↓
Enumerate SMB shares
    ↓
Check permissions
    ↓
Prioritize interesting shares
    ↓
Search filenames
    ↓
Search file contents
    ↓
Validate findings
    ↓
Reuse / crack / authenticate
```

## 1. Search Patterns

### Content keywords

```
passw
password
passwd
user
username
credential
cred
token
secret
key
```

Also consider target-specific/domain strings:

```
DOMAIN\
DOMAIN.LOCAL
company-specific usernames
local-language terms
```

Example:

```
INLANEFREIGHT\
```

## 2. Interesting Extensions

```
.ini
.cfg
.conf
.config
.env
.xml
.xlsx
.xls
.ps1
.bat
.cmd
```

Also investigate:

```
.txt
.csv
.json
.yml
.yaml
.py
.sh
.php
```

## 3. Interesting Filenames

```
config
configs
credential
credentials
password
passwords
passwd
secret
secrets
initial
backup
old
deploy
deployment
unattend
```

## 4. Windows Manual Search

List files:

```powershell
Get-ChildItem -Recurse \\SERVER\Share
```

Search contents:

```powershell
Get-ChildItem -Recurse \\SERVER\Share -File |
    Select-String -Pattern "password","passw","secret","token"
```

Target extensions:

```powershell
Get-ChildItem -Recurse \\SERVER\Share `
    -Include *.ini,*.cfg,*.xml,*.ps1,*.bat |
    Select-String -Pattern "password","user","secret"
```

Core idea:

```
Enumerate → filter → inspect
```

## 5. Snaffler

Windows/C# automated network-share hunting.

Basic:

```
Snaffler.exe -s
```

Capabilities:

- AD computer discovery
- DFS/share discovery
- Readable-share discovery
- File hunting
- Credential/secret pattern matching

Useful options from the HTB module:

```
-u
    search for references to AD users

-i / -n
    control share targets/search scope
```

Check installed version:

```
Snaffler.exe -h
```

Important:

```
Automated match ≠ confirmed credential
```

Always validate findings manually.

## 6. PowerHuntShares

PowerShell-based SMB/share enumeration and hunting.

Example:

```powershell
Invoke-HuntSMBShares `
    -Threads 100 `
    -OutputDirectory C:\Users\Public
```

Can:

- Discover domain computers
- Check SMB/445
- Enumerate shares
- Enumerate permissions
- Identify excessive access
- Identify readable/writable shares
- Search files
- Generate HTML report
- Generate CSV data

Output may include:

- HTML summary
- Detailed CSV results
- Timelines
- Share permissions
- Interesting files
- Potential secrets

Important:

```
Large environments can take significant time.
```

## 7. MANSPIDER

Linux-friendly SMB file hunting.

Docker approach:

```bash
docker run --rm \
  -v ./manspider:/root/.manspider \
  blacklanternsecurity/manspider \
  <TARGET> \
  -c 'passw' \
  -u '<USER>' \
  -p '<PASSWORD>'
```

Example:

```bash
docker run --rm \
  -v ./manspider:/root/.manspider \
  blacklanternsecurity/manspider \
  10.129.234.121 \
  -c 'passw' \
  -u 'mendres' \
  -p 'Inlanefreight2025!'
```

Important concepts:

```
-c
    search file contents

-u
    SMB username

-p
    SMB password

/root/.manspider/loot
    downloaded matching files
```

Check current options:

```bash
docker run --rm blacklanternsecurity/manspider --help
```

## 8. NetExec Share Spidering

Search an SMB share for matching file contents:

```bash
nxc smb <TARGET> \
    -u <USER> \
    -p '<PASSWORD>' \
    --spider <SHARE> \
    --content \
    --pattern "passw"
```

Example:

```bash
nxc smb 10.129.234.121 \
    -u mendres \
    -p 'Inlanefreight2025!' \
    --spider IT \
    --content \
    --pattern "passw"
```

Concept:

```
SMB authentication
      ↓
--spider SHARE
      ↓
crawl share
      ↓
--content
      ↓
search file contents
      ↓
--pattern
      ↓
match chosen string
```

Check local syntax:

```bash
nxc smb -h
```

## 9. Prioritizing Shares

Prefer investigating shares associated with:

```
IT
Infrastructure
Engineering
Deployment
Software
Backups
Scripts
Administration
```

over large low-value collections such as:

```
Photos
Marketing media
Public images
```

Think:

```
Business role
    ↓
Likely credentials
    ↓
Likely files
```

## 10. Validate Findings

Automated tools generate false positives.

For every finding:

```
Tool match
    ↓
Open file
    ↓
Understand context
    ↓
Is it real?
    ↓
Is it current?
    ↓
What account/service does it belong to?
    ↓
Can it actually authenticate?
```

Do not assume:

```
password-like string = valid password
```

## 11. Important Windows Artifacts

Recognize:

```
unattend.xml
```

Potential locations include:

```
C:\Windows\Panther\
```

or deployment/admin shares.

Also search for:

```
*.ps1
*.bat
*.cmd
*.ini
*.config
*.xml
```

## 12. Useful SMB Enumeration Before Hunting

From Linux:

```bash
nxc smb <TARGET> -u <USER> -p '<PASSWORD>' --shares
```

Then identify:

```
READ
WRITE
```

Prioritize:

```
READ → content hunting
WRITE → potentially interesting modification/privilege paths
```

Remember:

```
WRITE access ≠ automatic code execution
```

## 13. Credential Types

A share may contain:

```
Plaintext password
       ↓
use directly where authorized

Hash
       ↓
identify format → crack if appropriate

SSH private key
       ↓
authenticate directly

API token
       ↓
authenticate to supported service

Configuration file
       ↓
extract service credentials

Password spreadsheet
       ↓
validate account/password pair
```

## 14. Complete Workflow

```
Initial foothold
      ↓
Identify current user
      ↓
Identify domain
      ↓
Enumerate computers
      ↓
Enumerate SMB shares
      ↓
Check share permissions
      ↓
Prioritize IT/admin/deployment/etc.
      ↓
Search filenames
      ↓
Search file contents
      ↓
Review automated findings
      ↓
Validate credentials
      ↓
Authenticate to another service
      ↓
Privilege escalation / lateral movement
```

## 15. Tool Selection

```
Windows / AD environment
    ↓
Snaffler
PowerHuntShares

Linux
    ↓
MANSPIDER
NetExec

Small targeted search
    ↓
PowerShell / manual SMB enumeration
```

## Golden Rules

1. A network share is effectively a remote filesystem.

2. Enumerate permissions before searching contents.

3. IT/admin/deployment shares are usually more relevant than generic media shares.

4. Search both filenames and file contents.

5. Search for secrets, tokens, keys, and credentials—not only "password".

6. Use organization/domain-specific strings.

7. Automated tools produce false positives; validate every important finding.

8. A readable file may contain credentials for a completely different system.

9. A network-share credential can turn a local foothold into lateral movement.

10. Don't scan everything blindly—use the machine's role and the share's purpose to guide the hunt.
