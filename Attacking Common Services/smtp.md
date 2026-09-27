# Attacking Email Services — CPTS Cheatsheet

## 1. Email Service Basics

Email infrastructure mainly uses three protocols:

| Protocol        |  Port | Purpose                       |
| --------------- | ----: | ----------------------------- |
| SMTP            |  `25` | Sending/relaying email        |
| SMTP + STARTTLS | `587` | Authenticated mail submission |
| SMTP over TLS   | `465` | Encrypted SMTP                |
| POP3            | `110` | Downloading email             |
| POP3S           | `995` | Encrypted POP3                |
| IMAP            | `143` | Accessing/managing mailbox    |
| IMAPS           | `993` | Encrypted IMAP                |

### SMTP vs POP3 vs IMAP

```text
Sender
  │
  │ SMTP
  ↓
Mail Server
  │
  ├── SMTP → another mail server
  │
  └── IMAP/POP3 → recipient's mail client
```

### POP3

Usually downloads messages locally and may remove them from the server.

### IMAP

Normally keeps messages on the server and synchronizes them across devices.

**Pentesting focus:**

```text
DNS/MX
  ↓
SMTP/IMAP/POP3 enumeration
  ↓
Username enumeration
  ↓
Authentication attacks
  ↓
Misconfiguration
  ↓
Known vulnerabilities
```

---

# 2. Start with MX Records

Before attacking a mail server, determine **where the domain's email is hosted**.

### Host

```bash
host -t MX <DOMAIN>
```

Example:

```bash
host -t MX hackthebox.eu
```

### Dig

```bash
dig MX <DOMAIN>
```

Cleaner:

```bash
dig MX <DOMAIN> | grep "MX" | grep -v ";"
```

Example:

```text
inlanefreight.com.  300  IN  MX  10 mail1.inlanefreight.com.
```

Then resolve the mail server:

```bash
host -t A mail1.inlanefreight.com
```

### Why MX matters

The MX record tells you:

```text
Domain
   ↓
Mail provider/server
   ↓
What to enumerate next
```

You may discover:

```text
Google Workspace
Microsoft 365
Zoho
Custom mail server
```

Cloud-hosted email and self-hosted mail servers require different enumeration approaches.

---

# 3. Mail Service Enumeration

For a custom mail server, scan the common ports:

```bash
nmap -Pn -sV -sC -p25,110,143,465,587,993,995 <TARGET>
```

Look for:

```text
25/tcp   smtp
110/tcp  pop3
143/tcp  imap
465/tcp  smtps
587/tcp  submission
993/tcp  imaps
995/tcp  pop3s
```

### Why `-sV -sC`?

You want:

```text
Service
Version
SMTP capabilities
Authentication methods
Other useful banner information
```

Example:

```text
25/tcp open smtp Postfix smtpd
|_smtp-commands: mail1.inlanefreight.htb,
| PIPELINING, SIZE, VRFY, ETRN, ...
```

**Important clue:** `VRFY`, `EXPN`, or similar capabilities can indicate potential username-enumeration opportunities.

---

# 4. SMTP Enumeration

SMTP can sometimes reveal valid usernames.

The main commands to remember:

```text
VRFY
EXPN
RCPT TO
```

---

## VRFY

Checks whether a mailbox/user exists.

Connect:

```bash
telnet <TARGET> 25
```

Then:

```text
VRFY root
```

Possible valid response:

```text
252 2.0.0 root
```

Invalid:

```text
550 5.1.1 User unknown
```

### Mental model

```text
VRFY username
      ↓
SMTP server checks account
      ↓
Valid / Invalid response
```

---

# 5. EXPN

`EXPN` can expand a mailing list or alias.

```text
EXPN support-team
```

Possible result:

```text
250 carol@inlanefreight.htb
250 elisa@inlanefreight.htb
```

### Why EXPN can be worse

A single alias may reveal **multiple users**.

```text
support-team
      ↓
 ┌────┴────┐
 ↓         ↓
carol     elisa
```

---

# 6. RCPT TO

`RCPT TO` specifies the recipient during SMTP mail delivery.

It can sometimes be abused for username enumeration.

Example:

```bash
telnet <TARGET> 25
```

Then:

```text
MAIL FROM:test@htb.com
```

```text
RCPT TO:john
```

Possible valid response:

```text
250 2.1.5 john... Recipient ok
```

Invalid:

```text
550 5.1.1 john... User unknown
```

### Mental model

```text
MAIL FROM
    ↓
RCPT TO
    ↓
Server checks recipient
    ↓
Different response
    ↓
Potential username enumeration
```

---

# 7. POP3 USER Enumeration

POP3 may also expose whether a username exists.

Connect:

```bash
telnet <TARGET> 110
```

Then:

```text
USER john
```

Possible responses:

```text
+OK
```

versus:

```text
-ERR
```

If the implementation distinguishes valid/invalid users, this can be used for enumeration.

---

# 8. Automating SMTP Enumeration

Use:

```bash
smtp-user-enum
```

### RCPT mode

```bash
smtp-user-enum -M RCPT -U userlist.txt -D <DOMAIN> -t <TARGET>
```

### VRFY

```bash
smtp-user-enum -M VRFY -U userlist.txt -D <DOMAIN> -t <TARGET>
```

### EXPN

```bash
smtp-user-enum -M EXPN -U userlist.txt -D <DOMAIN> -t <TARGET>
```

### Modes

```text
-M VRFY  → verify users
-M EXPN  → expand aliases/lists
-M RCPT  → test recipients
```

### Workflow

```text
userlist.txt
     ↓
smtp-user-enum
     ↓
SMTP server
     ↓
valid usernames
```

---

# 9. Cloud Email Enumeration

Cloud providers implement email authentication differently.

Common examples:

```text
Microsoft 365
Google Workspace
Zoho
```

First determine the provider through:

```bash
dig MX <DOMAIN>
```

For Microsoft 365, HTB demonstrates:

```text
o365spray
```

### Validate O365

```bash
python3 o365spray.py --validate --domain <DOMAIN>
```

### Enumerate users

```bash
python3 o365spray.py --enum -U users.txt --domain <DOMAIN>
```

Concept:

```text
Domain
  ↓
Identify provider
  ↓
Provider-specific enumeration
  ↓
Valid accounts
```

**Important:** Cloud authentication changes frequently, so tool behavior can become outdated. Understanding the underlying authentication flow is more important than memorizing a tool.

---

# 10. Password Attacks

Once valid usernames are identified, authentication attacks may become possible.

Common targets:

```text
SMTP
POP3
IMAP
Cloud email authentication
```

For a lab/HTB target, Hydra can be used against supported protocols.

### POP3

```bash
hydra -L users.txt -p 'PASSWORD' -f <TARGET> pop3
```

### General concept

```text
Valid usernames
      +
Candidate password
      ↓
Authentication service
      ↓
Valid credential?
```

### Password spraying vs brute force

**Password spraying:**

```text
many users
    +
one/common password
```

**Brute force:**

```text
one user
    +
many passwords
```

### Important

Mail services and cloud providers commonly implement:

* Rate limiting
* Account lockouts
* MFA
* Detection
* IP blocking

So understand the authentication mechanism before blindly attacking it.

---

# 11. Microsoft 365 Password Spraying

HTB demonstrates:

```bash
python3 o365spray.py --spray \
-U usersfound.txt \
-p 'PASSWORD' \
--count 1 \
--lockout 1 \
--domain <DOMAIN>
```

Concept:

```text
usersfound.txt
      ↓
one password
      ↓
Microsoft 365
      ↓
authentication responses
```

The important idea is **low-volume spraying while accounting for lockout policies**, not simply trying huge numbers of passwords.

---

# 12. SMTP Open Relay

An **open relay** is an SMTP server that allows unauthorized users to relay mail through it.

Normal:

```text
Authorized sender
       ↓
SMTP server
       ↓
Recipient
```

Open relay:

```text
Anyone
  ↓
SMTP server
  ↓
External/internal recipient
```

### Why it matters

Potential abuse includes:

* Spam
* Sender spoofing
* Phishing
* Reputation damage
* Bypassing restrictions depending on configuration

---

# 13. Test for Open Relay

Nmap provides:

```bash
nmap -Pn -p25 --script smtp-open-relay <TARGET>
```

Possible result:

```text
smtp-open-relay: Server is an open relay
```

### Mental model

```text
Unauthenticated attacker
          ↓
      SMTP server
          ↓
      accepts relay
          ↓
       recipient
```

**Key distinction:**

An SMTP server accepting mail for its own domain does **not** automatically mean it is an open relay.

The problem is unauthorized **relaying to destinations the server should not permit**.

---

# 14. SMTP Testing with Swaks

`swaks` is useful for manually testing SMTP behavior.

Basic:

```bash
swaks \
--from test@example.com \
--to user@example.com \
--server <SMTP_SERVER>
```

With subject/body:

```bash
swaks \
--from test@example.com \
--to user@example.com \
--header 'Subject: Test' \
--body 'Test message' \
--server <SMTP_SERVER>
```

### Why Swaks is useful

It lets you control SMTP fields such as:

```text
MAIL FROM
RCPT TO
Subject
Body
Headers
Authentication
TLS
```

So it is useful for validating SMTP behavior after enumeration.

---

# 15. Known SMTP Vulnerabilities

## OpenSMTPD CVE-2020-7247

HTB covers:

```text
OpenSMTPD
CVE-2020-7247
```

The vulnerability affected certain OpenSMTPD versions and could allow **unauthenticated remote code execution**.

### Core concept

The vulnerable processing of SMTP sender input allowed command injection through specially crafted input.

Simplified:

```text
SMTP sender input
       ↓
Vulnerable processing
       ↓
Special character/input
       ↓
Command injection
       ↓
OS command execution
```

The key lesson is not memorizing the exploit string.

Understand:

```text
Network input
      ↓
Application parsing
      ↓
Unsafe command construction
      ↓
Shell interpretation
      ↓
RCE
```

---

# 16. OpenSMTPD Attack Model

Think about vulnerabilities using:

```text
SOURCE
  ↓
PROCESS
  ↓
PRIVILEGES
  ↓
DESTINATION
```

For CVE-2020-7247:

```text
SMTP input
    ↓
OpenSMTPD processing
    ↓
Service/process privileges
    ↓
Command execution
```

This is a useful general methodology for understanding service vulnerabilities.

---

# 17. Email Attack Surface

When you find a mail server, think in layers:

```text
                 MAIL SERVER
                      │
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
       SMTP          POP3          IMAP
        │             │             │
        ↓             ↓             ↓
   VRFY/EXPN       USER enum    Authentication
   RCPT TO
        │             │             │
        └─────────────┼─────────────┘
                      ↓
                Valid accounts
                      ↓
               Password attacks
                      ↓
                 Mailbox access
```

Then separately check:

```text
SMTP
 ├── Open relay
 ├── Weak authentication
 ├── User enumeration
 └── Known vulnerabilities
```

---

# 18. Full Enumeration Workflow

Use this during CPTS/HTB labs:

```text
                DOMAIN
                   │
                   ↓
               MX RECORD
                   │
                   ↓
        Identify mail provider
                   │
          ┌────────┴────────┐
          ↓                 ↓
      Cloud email       Custom server
          │                 │
          ↓                 ↓
 Provider-specific       Nmap
 enumeration               │
          │          ┌──────┼──────┐
          │          ↓      ↓      ↓
          │         SMTP   POP3   IMAP
          │          │      │      │
          │          └──────┼──────┘
          │                 ↓
          │        Username enumeration
          │                 ↓
          │        Valid usernames
          │                 ↓
          │       Authentication tests
          │                 ↓
          └──────────→ Mailbox access
```

---

# 19. Practical Command Workflow

```bash
# 1. Find mail servers
dig MX <DOMAIN>

# 2. Resolve the mail server
host -t A <MAIL_SERVER>

# 3. Scan mail services
nmap -Pn -sV -sC \
-p25,110,143,465,587,993,995 \
<TARGET>

# 4. Test SMTP manually
telnet <TARGET> 25

# 5. Test SMTP username enumeration
smtp-user-enum -M VRFY -U users.txt -D <DOMAIN> -t <TARGET>

# 6. Try RCPT enumeration if supported
smtp-user-enum -M RCPT -U users.txt -D <DOMAIN> -t <TARGET>

# 7. Test POP3
telnet <TARGET> 110

# 8. Test for open relay
nmap -Pn -p25 --script smtp-open-relay <TARGET>

# 9. SMTP interaction/testing
swaks --from test@example.com \
--to user@example.com \
--server <TARGET>
```

---

# 20. Mental Model

When you see **email services**, immediately think:

```text
1. WHERE?
   ↓
MX records

2. WHAT?
   ↓
SMTP / POP3 / IMAP

3. WHO?
   ↓
Username enumeration

4. CAN I AUTH?
   ↓
Credential testing

5. CAN I RELAY?
   ↓
Open relay

6. IS IT VULNERABLE?
   ↓
Version / known CVEs

7. WHAT DID I GET?
   ↓
Mailbox / credentials / internal information
```

---

# 21. What to Remember for CPTS

| Technique             | What you're looking for  | Main tool                         |
| --------------------- | ------------------------ | --------------------------------- |
| MX enumeration        | Mail infrastructure      | `dig`, `host`                     |
| Service enumeration   | SMTP/POP3/IMAP           | `nmap`                            |
| SMTP user enumeration | Valid accounts           | `VRFY`, `EXPN`, `RCPT TO`         |
| Automated SMTP enum   | Valid accounts           | `smtp-user-enum`                  |
| POP3 enumeration      | Valid accounts           | `USER`                            |
| Cloud enumeration     | Valid cloud accounts     | Provider-specific tools           |
| Password attack       | Valid credentials        | `hydra` / provider-specific tools |
| Open relay            | Unauthorized SMTP relay  | `nmap smtp-open-relay`            |
| SMTP testing          | Mail behavior            | `swaks`                           |
| Known vulnerabilities | Vulnerable mail software | Version/CVE research              |

---

# 22. Golden Rules

```text
MX
 ↓
Find the mail server/provider

25
 ↓
SMTP

110
 ↓
POP3

143
 ↓
IMAP

465
 ↓
SMTPS

587
 ↓
SMTP submission / STARTTLS

993
 ↓
IMAPS

995
 ↓
POP3S
```

```text
VRFY
 ↓
Can reveal whether a user exists

EXPN
 ↓
Can reveal members of aliases/lists

RCPT TO
 ↓
Can sometimes reveal valid recipients

POP3 USER
 ↓
Can sometimes reveal valid accounts
```

```text
Open relay
 ↓
Unauthenticated/unauthorized mail relay
 ↓
Potential spam/phishing/spoofing abuse
```

```text
Known vulnerable SMTP software
 ↓
Identify exact version
 ↓
Map to CVEs
 ↓
Understand vulnerability mechanism
 ↓
Validate safely in the lab
```

## Most important CPTS takeaway

**Don't start by attacking SMTP blindly.**

Start with:

```text
DOMAIN
  ↓
MX
  ↓
Mail provider/server
  ↓
Ports + versions
  ↓
SMTP/POP3/IMAP capabilities
  ↓
Username enumeration
  ↓
Authentication
  ↓
Misconfigurations
  ↓
Known vulnerabilities
```

The core skill is understanding **what the mail server exposes and why each response matters**. Once you have valid usernames, credentials, mailbox access, or an SMTP misconfiguration, the email service can become a path to further internal enumeration and compromise.
