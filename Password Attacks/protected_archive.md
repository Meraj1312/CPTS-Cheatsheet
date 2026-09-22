# Cracking Protected Archives — CPTS Cheatsheet

**Core idea:** Identify protected container → determine protection mechanism → extract crackable data or test candidates directly → recover password → extract/unlock.

---

## 1. General Workflow

```
Protected archive/container
          ↓
Identify format
          ↓
Identify protection mechanism
          ↓
2john / format-specific method
          ↓
Crack/test candidates offline
          ↓
Password
          ↓
Extract / unlock / mount
```

> Not every protected format uses a 2john converter.

---

## 2. Identify Actual File Type

```bash
file <file>
```

**Example:**

```bash
file GZIP.gzip
```

Possible result:

```
openssl enc'd data with salted password
```

> Filename extension ≠ guaranteed actual format.

---

## 3. ZIP Cracking

```bash
zip2john ZIP.zip > zip.hash
john --wordlist=rockyou.txt zip.hash
john zip.hash --show
```

**Workflow:**

```
ZIP.zip → zip2john → zip.hash → John → password
```

`zip2john` extracts password-verification data — it does **not** recover the password.

---

## 4. OpenSSL-Encrypted GZIP

If `file` shows:

```
openssl enc'd data with salted password
```

**Test candidates directly:**

```bash
for i in $(cat rockyou.txt); do
  openssl enc -aes-256-cbc -d -in GZIP.gzip -k $i 2>/dev/null | tar xz
done
```

**Concept:**

```
for each candidate → OpenSSL decrypt → gzip decompress → tar extract
→ successful extraction = correct password
```

**Commands:**

| Flag | Meaning |
| --- | --- |
| `openssl enc -aes-256-cbc -d` | Decrypt using AES-256-CBC |
| `-k $i` | Password candidate |
| `2>/dev/null` | Suppress errors |
| `tar xz` | Extract gzip-compressed TAR |

**Errors like these are expected on wrong candidates:**

```
gzip: stdin: not in gzip format
tar: Child returned status 1
```

Valid extraction = success signal.

---

## 5. BitLocker

**Extract hashes:**

```bash
bitlocker2john -i Backup.vhd > backup.hashes
grep "bitlocker\$0" backup.hashes > backup.hash
cat backup.hash
```

**Crack with Hashcat (mode 22100):**

```bash
hashcat -a 0 -m 22100 backup.hash /usr/share/wordlists/rockyou.txt
```

| Flag | Meaning |
| --- | --- |
| `-a 0` | Dictionary attack |
| `-m 22100` | BitLocker |

**Why slow (~25 H/s):** password-verification is intentionally expensive (KDF). Compare cracking speeds only when algorithms/work factors are comparable.

**Password vs Recovery Key:**

- Password → human-generated → potentially guessable
- Recovery key → long/random → impractical to brute-force

**Success output:**

```
Status...........: Cracked
Hash.Mode........: 22100 (BitLocker)
Recovered........: 1/1
```

---

## 6. BitLocker Unlock — Linux

```bash
# Install
sudo apt-get install dislocker

# Mount points
sudo mkdir -p /media/bitlocker
sudo mkdir -p /media/bitlockermount

# Expose VHD as loop device
sudo losetup -f -P Backup.vhd

# Unlock
sudo dislocker /dev/loop0p2 -u<password> -- /media/bitlocker

# Mount
sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount

# Access
cd /media/bitlockermount && ls -la
```

> `/dev/loop0p2` is example-specific — partition number varies.

**Unmount:**

```bash
sudo umount /media/bitlockermount
sudo umount /media/bitlocker
```

**Windows:** Mount VHD → BitLocker volume appears → enter recovered password → browse files.

---

## 7. Protection Types Comparison

| Format | Method |
| --- | --- |
| ZIP | `zip2john` → John |
| OpenSSL-encrypted | Test candidates with `openssl` → validate decrypted output |
| BitLocker | `bitlocker2john` → Hashcat/John → unlock volume |

---

## 8. Decision Tree

```
Found protected file/container
              ↓
          file <file>
              ↓
       What is the format?
              │
      ┌───────┼─────────┐
      ↓       ↓         ↓
     ZIP    OpenSSL   BitLocker
      ↓       ↓         ↓
 zip2john   direct   bitlocker2john
      ↓      testing      ↓
    John    + validate  Hashcat/John
      ↓                   ↓
 password              password
      ↓                   ↓
 extract               unlock/mount
```

---

## 9. Must-Remember Commands

```bash
# Identify
file <file>

# ZIP
zip2john ZIP.zip > zip.hash
john --wordlist=rockyou.txt zip.hash
john zip.hash --show

# BitLocker
bitlocker2john -i Backup.vhd > backup.hashes
grep "bitlocker\$0" backup.hashes > backup.hash
hashcat -a 0 -m 22100 backup.hash rockyou.txt

# dislocker
sudo apt-get install dislocker
sudo mkdir -p /media/bitlocker /media/bitlockermount
sudo losetup -f -P Backup.vhd
sudo dislocker /dev/loop0p2 -u<password> -- /media/bitlocker
sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount
sudo umount /media/bitlockermount && sudo umount /media/bitlocker
```

---

## 10. Memory Rules

```
ZIP:        zip2john → John
BitLocker:  bitlocker2john → Hashcat/John
OpenSSL:    candidate testing → validate successful extraction
file:       Trust contents more than filename
```

---

## 11. One-Line Mental Model

> Identify the real protection mechanism first — then either extract a crackable representation or test candidates directly — and use the recovered secret to unlock the data.
