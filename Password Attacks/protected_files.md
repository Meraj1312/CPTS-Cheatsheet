# Cracking Protected Files — CPTS Cheatsheet

**Core idea:** Find protected file → identify → extract crackable data → crack offline → use recovered password.

---

## 1. Core Workflow

```
Find file → Identify type → Check protection
→ <format>2john → John/Hashcat → Recovered password
```

---

## 2. Hash vs Encrypted File

```
Hash:      candidate → hash(candidate) → compare
Protected: candidate password → derive/test → compare with file metadata
```

---

## 3. Find Protected Files

```bash
# By extension
find / -name '*.pdf' 2>/dev/null
find / \( -name '*.pdf' -o -name '*.docx' -o -name '*.xlsx' \) 2>/dev/null

# By content (SSH keys)
grep -rni "BEGIN .*PRIVATE KEY" /home 2>/dev/null
grep -rnE '^\-{5}BEGIN [A-Z0-9]+ PRIVATE KEY\-{5}$' /* 2>/dev/null

# Identify real type
file <filename>
```

> Extension ≠ proof of encryption. Always verify with `file`.

---

## 4. SSH Key Passphrase Check

```bash
ssh-keygen -yf <private_key>
```

- Works directly → no passphrase
- Prompts passphrase → protected

Old PEM encrypted keys may show:
```
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,...
```
Modern OpenSSH keys may not. Always test with `ssh-keygen -yf`.

---

## 5. Find 2john Tools

```bash
locate *2john*
```

| File | Converter |
| --- | --- |
| SSH private key | ssh2john |
| ZIP | zip2john |
| RAR | rar2john |
| PDF | pdf2john |
| Office | office2john |
| KeePass | keepass2john |
| PuTTY key | putty2john |
| PFX/PKCS#12 | pfx2john |
| WPA | wpa2john |

---

## 6. Crack Workflow (Universal)

```bash
<converter> <file> > file.hash
john --wordlist=rockyou.txt file.hash
john file.hash --show
```

**Examples:**

```bash
# SSH
ssh2john.py SSH.private > ssh.hash
john --wordlist=rockyou.txt ssh.hash
john ssh.hash --show

# Office
office2john.py Protected.docx > protected.hash
john --wordlist=rockyou.txt protected.hash
john protected.hash --show

# PDF
pdf2john.py PDF.pdf > pdf.hash
john --wordlist=rockyou.txt pdf.hash
john pdf.hash --show
```

---

## 7. When rockyou Fails

Ask:
- Owner? Company? Project?
- Password policy? Years/months?
- Useful OSINT?

Then:
```
OSINT → custom wordlist → rules → John
```

---

## 8. Why It Matters

Recovered passwords can expose:
- Sensitive documents, credentials, hostnames, URLs
- Additional attack paths

---

## 9. Limitations

Success depends on: password quality + candidate quality + KDF strength + hardware + time.

Strong random password + strong KDF = potentially impractical.

---

## 10. Must-Remember Commands

```bash
find / -name '*.pdf' 2>/dev/null
file <file>
grep -rni "BEGIN .*PRIVATE KEY" /home 2>/dev/null
ssh-keygen -yf <private_key>
locate *2john*
ssh2john.py key > key.hash && john --wordlist=rockyou.txt key.hash && john key.hash --show
office2john.py file.docx > office.hash && john --wordlist=rockyou.txt office.hash
pdf2john.py file.pdf > pdf.hash && john --wordlist=rockyou.txt pdf.hash
```

---

## 11. Universal Memory Block

```
1. Find it
2. Identify it
3. Check if password-protected
4. Find matching 2john converter
5. Extract crackable representation
6. Wordlist / custom wordlist / rules
7. John
8. john --show
9. Use recovered password
```

---

## 12. One-Line Mental Model

> Don't crack the file directly: identify the protection, extract the data John can attack, then crack the password offline.
