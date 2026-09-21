# John the Ripper (JtR)

> **Core idea:** John guesses password candidates, hashes them using the target format, and compares the result against the target. It does not "decrypt" normal password hashes.

---

## 1. Basic Mental Model

```text
Target hash
    ↓
Identify format
    ↓
Choose attack mode
    ↓
Generate candidates
    ↓
Hash candidates
    ↓
Compare
    ↓
Match → password recovered
```

---

## 2. Basic Syntax

```bash
john [options] <hash_file>
```

Useful general form:

```bash
john --format=<format> --wordlist=<wordlist> <hash_file>
```

---

# 3. Cracking Modes

## Single Crack Mode

Uses information associated with the account such as:

* username
* full name / GECOS
* home directory
* other account-related strings

```bash
john --single passwd
```

Good for:

```text
Linux account credentials
```

Mental model:

```text
Account information
      ↓
Candidate generation
      ↓
Rules/mutations
      ↓
Hash testing
```

---

## Wordlist Mode

Dictionary attack:

```bash
john --wordlist=<wordlist> <hash_file>
```

Example:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

Use when:

```text
Common passwords are likely
A suitable wordlist exists
```

---

## Wordlist + Rules

Generate mutations from wordlist entries:

```bash
john --wordlist=<wordlist> --rules <hash_file>
```

Example:

```bash
john --wordlist=words.txt --rules hashes.txt
```

Useful transformations may include:

```text
password
Password
PASSWORD
password1
Password1
Password1!
```

Mental model:

```text
Wordlist
   ↓
Rules
   ↓
More candidates
   ↓
Hash testing
```

---

## Incremental Mode

Brute-force-style statistical candidate generation:

```bash
john --incremental <hash_file>
```

Uses a statistical/Markov-based model to prioritize likely candidates.

More exhaustive, but potentially very expensive.

Think:

```text
more search space
      =
more time/compute
```

Configuration:

```bash
grep '# Incremental modes' -A 100 /etc/john/john.conf
```

---

# 4. Choosing an Attack

General strategy:

```text
Have useful target information?
        ↓
   Try --single

Have a good wordlist?
        ↓
   Try --wordlist

Need mutations?
        ↓
   Add --rules

Need a broader/exhaustive search?
        ↓
   Consider --incremental
```

Don't default to brute force without using information you already have.

---

# 5. Hash Identification

Use:

```bash
hashid <hash>
```

Include John formats:

```bash
hashid -j <hash>
```

Example:

```bash
hashid -j 193069ceb0461e1d40d216e32c79c704
```

Important:

```text
hashID output = possible formats
```

It does NOT always prove the exact format.

Use:

```text
Hash structure
+
Source/context
+
Application/system
```

to determine the most likely format.

Useful references:

```text
JtR sample hashes
PentestMonkey JtR hash format cheat sheet
```

---

# 6. Specify a Hash Format

Syntax:

```bash
john --format=<format> <hash_file>
```

Examples:

```bash
john --format=raw-md5 hashes.txt
john --format=nt hashes.txt
john --format=sha512crypt hashes.txt
john --format=raw-sha1 hashes.txt
john --format=raw-sha256 hashes.txt
```

List supported formats:

```bash
john --list=formats
```

---

# 7. Common JtR Formats

| Format       | Example meaning              |
| ------------ | ---------------------------- |
| `raw-md5`    | Raw MD5                      |
| `raw-sha1`   | Raw SHA-1                    |
| `raw-sha256` | Raw SHA-256                  |
| `raw-sha512` | Raw SHA-512                  |
| `nt`         | NTLM/NT hash                 |
| `lm`         | LM hash                      |
| `netntlm`    | NetNTLM                      |
| `netntlmv2`  | NetNTLMv2                    |
| `krb5`       | Kerberos 5                   |
| `mscash`     | Domain Cached Credentials    |
| `mscash2`    | Domain Cached Credentials v2 |
| `mysql`      | MySQL password hash          |
| `mssql`      | Microsoft SQL password hash  |
| `zip`        | ZIP                          |
| `rar`        | RAR                          |
| `pdf`        | PDF                          |
| `ssh`        | SSH-related format           |

Don't memorize every JtR format. Learn the pattern:

```bash
john --format=<john-format>
```

---

# 8. Password-Protected Files

John uses `2john` utilities to convert protected files into John-compatible representations.

General pattern:

```bash
<tool> <file> > file.hash
```

Then:

```bash
john <options> file.hash
```

---

## Common 2john Tools

| Tool                    | Purpose                     |
| ----------------------- | --------------------------- |
| `pdf2john`              | PDF                         |
| `ssh2john`              | SSH private keys            |
| `rar2john`              | RAR                         |
| `zip2john`              | ZIP                         |
| `keepass2john`          | KeePass                     |
| `office2john`           | MS Office                   |
| `pfx2john`              | PKCS#12/PFX                 |
| `putty2john`            | PuTTY private keys          |
| `wpa2john`              | WPA/WPA2 material           |
| `hccap2john`            | WPA/WPA2 handshake captures |
| `truecrypt_volume2john` | TrueCrypt                   |
| `keychain2john`         | OS X Keychain               |

Find available utilities:

```bash
locate *2john*
```

or:

```bash
which zip2john
which ssh2john
which pdf2john
```

---

# 9. Common File-Cracking Workflows

## ZIP

```bash
zip2john secret.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
john --show zip.hash
```

---

## SSH Private Key

```bash
ssh2john id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
john --show id_rsa.hash
```

---

## PDF

```bash
pdf2john secret.pdf > pdf.hash
john --wordlist=/usr/share/wordlists/rockyou.txt pdf.hash
john --show pdf.hash
```

---

## KeePass

```bash
keepass2john database.kdbx > keepass.hash
john --wordlist=/usr/share/wordlists/rockyou.txt keepass.hash
john --show keepass.hash
```

---

# 10. Display Recovered Passwords

After cracking:

```bash
john --show <hash_file>
```

Example:

```bash
john --show hashes.txt
```

Use this to reliably retrieve cracked credentials.

---

# 11. Typical Workflow — Hash

```bash
# 1. Identify
hashid -j <hash>

# 2. Save hash
echo '<hash>' > hashes.txt

# 3. Crack using a wordlist
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt

# 4. Display results
john --show hashes.txt
```

With explicit format:

```bash
john --format=<format> \
     --wordlist=/usr/share/wordlists/rockyou.txt \
     hashes.txt
```

---

# 12. Typical Workflow — Protected File

```bash
# 1. Convert
<format>2john <file> > file.hash

# 2. Crack
john --wordlist=/usr/share/wordlists/rockyou.txt file.hash

# 3. Show result
john --show file.hash
```

Example:

```bash
zip2john backup.zip > backup.hash
john --wordlist=/usr/share/wordlists/rockyou.txt backup.hash
john --show backup.hash
```

---

# 13. Useful Commands

### Show supported formats

```bash
john --list=formats
```

### Show cracked passwords

```bash
john --show hashes.txt
```

### Single crack

```bash
john --single hashes.txt
```

### Wordlist

```bash
john --wordlist=wordlist.txt hashes.txt
```

### Wordlist + rules

```bash
john --wordlist=wordlist.txt --rules hashes.txt
```

### Incremental

```bash
john --incremental hashes.txt
```

### Explicit format

```bash
john --format=<format> hashes.txt
```

### Locate conversion utilities

```bash
locate *2john*
```

---

# 14. Important Concepts

## Hash vs Encryption

### Hash

```text
password
   ↓
hash function
   ↓
hash
```

Usually attacked by guessing:

```text
candidate
   ↓
hash(candidate)
   ↓
compare
```

### Encrypted/protected file

```text
file + password
      ↓
encryption
      ↓
protected file
```

For John:

```text
protected file
      ↓
2john utility
      ↓
John-compatible data
      ↓
John
```

John is therefore generally **cracking the password**, not magically reversing the hash/encryption.

---

# 15. Hash Identification Rule

Never rely only on appearance.

```text
Hash string
+
Length/structure
+
Prefix/salt
+
Source
+
Application
=
Best format hypothesis
```

Example:

```text
193069ceb0461e1d40d216e32c79c704
```

may produce multiple `hashID` possibilities.

Context determines which one is plausible.

---

# 16. Attack Strategy

Preferred reasoning:

```text
Target information
      ↓
--single
      ↓
Known wordlist
      ↓
--wordlist
      ↓
Mutations
      ↓
--rules
      ↓
Broader search
      ↓
--incremental
```

The exact order depends on the target and password assumptions.

---

# 17. Common Mistakes

### Wrong format

```bash
john --format=raw-md5 hashes.txt
```

when the target isn't raw MD5.

Result:

```text
failure / invalid cracking assumptions
```

---

### Poor wordlist

Correct hash format + terrible candidate list can still result in:

```text
no password found
```

---

### Brute force too early

Don't ignore information such as:

```text
username
real name
company
hostname
application
password policy
known leaked passwords
```

---

### Confusing 2john output with the original password

For example:

```bash
zip2john secret.zip > zip.hash
```

does NOT recover the ZIP password.

It only creates input John can attack.

---

# 18. CPTS/HTB Memory Block

```text
JOHN THE RIPPER

# Hash identification
hashid -j <hash>

# List formats
john --list=formats

# Single crack
john --single hashes.txt

# Wordlist
john --wordlist=rockyou.txt hashes.txt

# Wordlist + rules
john --wordlist=rockyou.txt --rules hashes.txt

# Incremental
john --incremental hashes.txt

# Explicit format
john --format=<format> hashes.txt

# Show cracked passwords
john --show hashes.txt

# Protected file → John
<format>2john file > file.hash
john --wordlist=rockyou.txt file.hash
john --show file.hash

# Find 2john tools
locate *2john*
```

---

## 19. One-Line Mental Model

> **Identify the hash → choose the attack → generate candidates → John hashes/tests them → `--show` gives you the recovered credential.**
