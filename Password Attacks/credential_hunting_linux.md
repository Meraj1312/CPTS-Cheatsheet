Here is the content formatted as a Markdown (`.md`) file:

```markdown
# Credential Hunting in Linux

## Core Concept

After gaining a Linux foothold, search for credentials before attempting complicated privilege escalation.

**Four major sources:**

- Files
- History
- Memory
- Key-rings

**Common findings:**

- Passwords
- Hashes
- SSH keys
- API keys
- Tokens
- Database credentials
- Cloud credentials
- Session credentials

---

## 1. Understand the Host First

```bash
whoami
id
hostname
uname -a
cat /etc/os-release
```

**Enumerate users:**

```bash
cat /etc/passwd
```

**Current shell/environment:**

```bash
env
printenv
```

**Think:**

- What is this machine?
- What service does it provide?
- Who uses it?
- What credentials would that role need?

---

## 2. Configuration Files

**Common extensions:**

```
.conf
.config
.cnf
.ini
.yaml
.yml
.json
.xml
.env
.php
```

**Search:**

```bash
find / -type f \( -name "*.conf" -o -name "*.config" -o -name "*.cnf" \) 2>/dev/null
```

**Search configuration contents:**

```bash
grep -RiE "user|username|password|passwd|pwd|credential|secret|token|api[_-]?key|connection" /etc /opt /var/www 2>/dev/null
```

**Useful credential keywords:**

```
password
passwd
pwd
username
user
credential
secret
token
api_key
apikey
private_key
connectionstring
client_secret
```

---

## 3. Databases

**Search:**

```bash
find / -type f \( -name "*.sql" -o -name "*.db" -o -name "*.sqlite" -o -name "*.sqlite3" \) 2>/dev/null
```

**Identify:**

```bash
file <database>
```

**SQLite:**

```bash
sqlite3 <database>
```

**Inside SQLite:**

```sql
.tables
.schema
```

**Look for tables/columns containing:**

- users
- credentials
- passwords
- accounts
- tokens
- secrets
- config

---

## 4. Notes / Documents

**Find text files:**

```bash
find /home -type f -name "*.txt" 2>/dev/null
```

**Find files without an extension:**

```bash
find /home -type f ! -name "*.*" 2>/dev/null
```

**Combined:**

```bash
find /home -type f \( -name "*.txt" -o ! -name "*.*" \) 2>/dev/null
```

**Look for names:**

```bash
find /home /opt /tmp /var/www -type f \
\( -iname "*pass*" -o -iname "*cred*" -o -iname "*secret*" -o -iname "*backup*" -o -iname "*token*" \) \
2>/dev/null
```

---

## 5. Scripts / Source Code

**Search common extensions:**

```bash
find / -type f \
\( -name "*.sh" -o -name "*.py" -o -name "*.pl" -o -name "*.php" \
-o -name "*.js" -o -name "*.go" -o -name "*.jar" \
-o -name "*.rb" -o -name "*.yml" -o -name "*.yaml" \) \
2>/dev/null
```

**Search scripts for secrets:**

```bash
grep -RiE "password|passwd|pwd|secret|token|api[_-]?key|private_key" \
/opt /var/www /home 2>/dev/null
```

**Typical findings:**

- Hardcoded passwords
- Database credentials
- API keys
- SSH credentials
- Cloud credentials
- Service accounts

---

## 6. Cronjobs

**System-wide:**

```bash
cat /etc/crontab
```

**Directories:**

```bash
ls -la /etc/cron.*
ls -la /etc/cron.d/
```

**Current user's cron:**

```bash
crontab -l
```

**Inspect referenced scripts:**

```bash
ls -l <script>
cat <script>
```

**Look for:**

- Credentials inside scripts
- Writable scripts executed by privileged users
- Writable directories containing scheduled scripts
- Sensitive backup commands

**Important chain:**

```
root cronjob
    ↓
script
    ↓
hardcoded password
```

or:

```
root cronjob
    ↓
writable script
    ↓
privilege escalation
```

---

## 7. Bash History

```bash
cat ~/.bash_history
tail ~/.bash_history
```

**Search:**

```bash
grep -iE "password|passwd|pwd|secret|token|ssh|mysql|psql|curl|wget|sudo" ~/.bash_history 2>/dev/null
```

**Other shell history:**

```
~/.zsh_history
~/.fish_history
```

**Also inspect:**

```
~/.bashrc
~/.bash_profile
~/.profile
```

**Potential findings:**

- Passwords
- SSH commands
- Database credentials
- API tokens
- Internal hosts
- Administrative commands

---

## 8. Logs

**Common files:**

Debian / Ubuntu

```
/var/log/auth.log
/var/log/syslog
```

RHEL / CentOS

```
/var/log/secure
```

**Other useful logs:**

```
/var/log/cron
/var/log/kern.log
/var/log/dmesg
/var/log/mail.log
```

**Search:**

```bash
grep -RiE "accepted|failed|failure|ssh|sudo|password changed|new user|delete user|COMMAND=" /var/log 2>/dev/null
```

**Useful for discovering:**

- Valid usernames
- SSH access
- Login activity
- Sudo usage
- Service activity
- Authentication failures

---

## 9. Environment Variables

Credentials may be passed through environment variables.

```bash
env
printenv
```

**Search:**

```bash
env | grep -iE "pass|pwd|secret|token|key|api|credential"
```

**Also search files/scripts for:**

```
export PASSWORD=
export API_KEY=
export TOKEN=
```

---

## 10. SSH Keys

**Search user directories:**

```bash
find /home /root -type f \
\( -name "id_rsa" -o -name "id_dsa" -o -name "id_ecdsa" -o -name "id_ed25519" \) \
2>/dev/null
```

**Search more broadly:**

```bash
find /home /root /opt /tmp -type f \( -iname "*id_rsa*" -o -iname "*.pem" -o -iname "*.key" \) 2>/dev/null
```

**Inspect:**

```bash
cat <private_key>
```

**Look for:**

```
-----BEGIN OPENSSH PRIVATE KEY-----
-----BEGIN RSA PRIVATE KEY-----
```

A private key may provide direct authentication without password cracking.

---

## 11. Memory — Mimipenguin

Requires appropriate privileges.

```bash
sudo python3 mimipenguin.py
```

**Concept:**

```
Active session/application
        ↓
credentials in memory/cache
        ↓
memory extraction
```

---

## 12. LaZagne

Broad credential-recovery tool.

**Examples from the HTB material:**

```bash
sudo python2.7 laZagne.py all
```

**More targeted:**

```bash
python3 laZagne.py browsers
```

**Help:**

```bash
python3 laZagne.py -h
```

**Potential sources include:**

- WiFi
- Keyrings
- Firefox
- Chromium
- SSH
- Git
- AWS
- Docker
- KeePass
- Shadow
- Environment variables
- Sessions

Tool syntax/version support can vary, so check:

```bash
laZagne.py -h
```

before relying on old module commands.

---

## 13. Firefox Credentials

**Firefox profile locations:**

```bash
ls -la ~/.mozilla/firefox/
```

**Typical files:**

```
logins.json
key4.db
cert9.db
```

**Inspect login records:**

```bash
cat ~/.mozilla/firefox/<profile>/logins.json | jq .
```

**Important:**

```
logins.json
    ≠ plaintext passwords

logins.json
    = encrypted login records
```

Firefox Decrypt can be used against an accessible Firefox profile:

```bash
python3 firefox_decrypt.py
```

Follow the tool's current usage/help for the installed version.

**Potential output:**

```
Website
Username
Password
```

---

## 14. Keyrings

**Common Linux credential stores:**

- GNOME Keyring
- KWallet
- Libsecret

**These may hold:**

- Application passwords
- Network credentials
- Browser/application secrets
- Authentication material

They are normally encrypted/protected rather than plaintext files.

---

## 15. Credential Hunting vs Dumping vs Cracking

```
Credential Hunting
    ↓
Search broadly for credentials

Credential Dumping
    ↓
Extract from known credential stores

Password Cracking
    ↓
Recover password from a hash/encrypted credential
```

**Examples:**

```
Find password in config
    → Credential Hunting

Extract from /etc/shadow
    → Credential Dumping

Crack /etc/shadow hash
    → Password Cracking

Find SSH private key
    → Credential Hunting
    → Direct authentication may be possible

Find API token
    → Credential Hunting
    → Use token where authorized
```

---

## 16. Practical Workflow

```
1. whoami / id
        ↓
2. Identify host role
        ↓
3. Enumerate users/software
        ↓
4. Search configs
        ↓
5. Search databases
        ↓
6. Search notes/documents
        ↓
7. Search scripts/source code
        ↓
8. Enumerate cronjobs
        ↓
9. Check shell history
        ↓
10. Inspect relevant logs
        ↓
11. Check environment variables
        ↓
12. Search SSH keys
        ↓
13. Check memory/session credentials
        ↓
14. Check keyrings/browser stores
        ↓
15. Reuse discovered credentials
```

---

## Golden Rule

> Don't guess credentials
> when the system may already contain them.
```
