# Protected File Transfers

> Replace `<KEY>`, `<FILE>`, `<PATH>`, and `<IP>` with your values.

**Note:** Don't exfiltrate real PII/financial/trade-secret data unless the client specifically asked for it. For DLP/egress testing, use dummy data that mimics what the client is protecting. Encrypt sensitive files (NTDS.dit, credential dumps, enum data) before any transfer that isn't already over SSH/SFTP/HTTPS.

## 1. AES File/String Encryption — Windows (PowerShell)

### Import the module

```
Import-Module .\Invoke-AESEncryption.ps1
```

### Encrypt a string

```
Invoke-AESEncryption -Mode Encrypt -Key "<KEY>" -Text "Secret Text"
```

### Decrypt a string

```
Invoke-AESEncryption -Mode Decrypt -Key "<KEY>" -Text "<BASE64_CIPHERTEXT>"
```

### Encrypt a file (outputs `<FILE>.aes`)

```
Invoke-AESEncryption -Mode Encrypt -Key "<KEY>" -Path .\<FILE>
```

### Decrypt a file (strips `.aes`)

```
Invoke-AESEncryption -Mode Decrypt -Key "<KEY>" -Path .\<FILE>.aes
```

> Uses AES-256-CBC, key derived via SHA-256 of `<KEY>`, IV prepended to ciphertext. Use a unique password per engagement — never reuse one client's key on another.

---

## 2. File Encryption — Linux (OpenSSL)

### Encrypt

```
openssl enc -aes256 -iter 100000 -pbkdf2 -in <FILE> -out <FILE>.enc
```

### Decrypt

```
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in <FILE>.enc -out <FILE>
```

> `-iter 100000 -pbkdf2` hardens key derivation against brute-force. Swap `-aes256` for another cipher from `openssl enc -ciphers` if needed.

---

## 3. Transfer the Encrypted File

Use any standard file-transfer method (see `windows_file_transfer.md` / `linux_file_transfer.md`) once the file is encrypted:

```
HTTP/HTTPS  → python3 -m http.server / uploadserver
SMB/445     → impacket-smbserver
SFTP/SSH    → scp / sftp
FTP/21      → pyftpdlib
```

Prefer an already-encrypted transport (SSH/SFTP/HTTPS) when available — file-level encryption is the fallback for when it isn't.

---

## Quick Picks

```
Windows target, PowerShell available  → Invoke-AESEncryption.ps1
Linux target / attacker box           → openssl enc -aes256 -pbkdf2
Always                                 → unique key per engagement, no key reuse
Transport                              → SSH/SFTP/HTTPS first, encrypt-then-transfer otherwise
```
