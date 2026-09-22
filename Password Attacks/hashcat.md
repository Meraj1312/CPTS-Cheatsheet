# Hashcat — CPTS Cheatsheet

**Core idea:** `-a` = attack strategy. `-m` = hash type/mode.

---

## 1. Core Syntax

```bash
hashcat -a <attack_mode> -m <hash_mode> <hashes> [wordlist/rule/mask/...]
```

**Example:**

```bash
hashcat -a 0 -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

**Meaning:**

- `-a 0` → dictionary attack
- `-m 0` → MD5

---

## 2. Attack Modes

### Dictionary

```bash
hashcat -a 0 -m <mode> <hashes> <wordlist>
```

**Example:**

```bash
hashcat -a 0 -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

**Mental model:**

```
wordlist
   ↓
candidate
   ↓
hash(candidate)
   ↓
compare
```

### Dictionary + Rules

```bash
hashcat -a 0 -m <mode> <hashes> <wordlist> -r <ruleset>
```

**Example:**

```bash
hashcat -a 0 -m 0 hashes.txt \
/usr/share/wordlists/rockyou.txt \
-r /usr/share/hashcat/rules/best64.rule
```

Rules transform base words into additional candidates.

```
password
   ↓
rules
   ↓
Password
password1
Password1!
P@ssword
...
```

**Common rules directory:**

```
/usr/share/hashcat/rules
```

**Inspect:**

```bash
ls -l /usr/share/hashcat/rules
```

**Useful ruleset from this module:**

```
best64.rule
```

### Mask Attack

```bash
hashcat -a 3 -m <mode> <hashes> '<mask>'
```

**Example:**

```bash
hashcat -a 3 -m 0 hashes.txt '?u?l?l?l?l?d?s'
```

**Mental model:**

```
Mask = known password structure
```

---

## 3. Hash Modes

Hashcat assigns numeric IDs to hash types.

**View them:**

```bash
hashcat --help
```

**Important examples:**

| Hashcat mode | Hash |
| --- | --- |
| 0 | MD5 |
| 100 | SHA1 |
| 1300 | SHA2-224 |
| 1400 | SHA2-256 |
| 10800 | SHA2-384 |
| 1700 | SHA2-512 |
| 1000 | NTLM |
| 500 | MD5 Crypt |

Don't memorize hundreds of IDs. Know how to find them.

---

## 4. Hash Identification

**Use:**

```bash
hashid -m '<hash>'
```

**Example:**

```bash
hashid -m '$1$FNr44XZC$wQxY6HHLrgrGX0e1195k.1'
```

**Possible result:**

```
MD5 Crypt [Hashcat Mode: 500]
```

**Then:**

```bash
-m 500
```

**Important:**

hashID result = possible format(s)

Use context/source to determine the most likely format.

---

## 5. Built-in Mask Character Sets

| Symbol | Charset |
| --- | --- |
| ?l | abcdefghijklmnopqrstuvwxyz |
| ?u | ABCDEFGHIJKLMNOPQRSTUVWXYZ |
| ?d | 0123456789 |
| ?h | 0123456789abcdef |
| ?H | 0123456789ABCDEF |
| ?s | Special characters |
| ?a | ?l?u?d?s |
| ?b | 0x00 - 0xff |

**Most important:**

- `?l` → lowercase
- `?u` → uppercase
- `?d` → digit
- `?s` → special
- `?a` → all standard character classes

---

## 6. Mask Examples

**Four lowercase letters**

```
'?l?l?l?l'
```

**Four digits**

```
'?d?d?d?d'
```

**Uppercase + lowercase + digits**

```
'?u?l?l?l?d?d'
```

**Uppercase + 4 lowercase + digit + symbol**

```
'?u?l?l?l?l?d?s'
```

**Command:**

```bash
hashcat -a 3 -m 0 hashes.txt '?u?l?l?l?l?d?s'
```

---

## 7. Custom Character Sets

Define custom sets with:

```
-1
-2
-3
-4
```

Reference them with:

```
?1
?2
?3
?4
```

**Mental model:**

```
-1 <custom charset>
      ↓
?1 in mask
```

Use custom sets to reduce unnecessary keyspace when you know something about the target.

---

## 8. Keyspace

Keyspace = total number of candidates represented by the attack.

**Examples:**

```
?d?d
```

= 10 × 10

= 100 candidates.

```
?l?l?l?l
```

= 26^4

= 456,976 candidates.

**General idea:**

```
more positions
+
larger character sets
=
larger keyspace
```

Always consider keyspace before launching a large mask attack.

---

## 9. Rules

Hashcat rules modify words from a base wordlist.

**Common location:**

```
/usr/share/hashcat/rules
```

**List rules:**

```bash
ls -l /usr/share/hashcat/rules
```

**Common examples:**

```
best64.rule
dive.rule
rockyou-30000.rule
leetspeak.rule
```

**Apply a ruleset:**

```bash
-r /usr/share/hashcat/rules/best64.rule
```

**Complete example:**

```bash
hashcat -a 0 -m 0 hashes.txt \
/usr/share/wordlists/rockyou.txt \
-r /usr/share/hashcat/rules/best64.rule
```

---

## 10. Hashcat Output

**Important fields:**

```
Status
Hash.Mode
Hash.Target
Guess.Base
Guess.Mod
Guess.Mask
Speed
Recovered
Progress
```

**Status**

```
Status...........: Cracked
```

Target recovered.

**Hash mode**

```
Hash.Mode........: 0 (MD5)
```

Confirms selected mode.

**Dictionary source**

```
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
```

**Rule source**

```
Guess.Mod........: Rules (/usr/share/hashcat/rules/best64.rule)
```

**Mask**

```
Guess.Mask.......: ?u?l?l?l?l?d?s
```

**Recovered**

```
Recovered........: 1/1
```

One out of one targets recovered.

**Progress**

```
Progress.........: 28672/14344385
```

Progress through the candidate space.

**Speed**

```
Speed.#1.........: 1706.6 kH/s
```

Hash calculations per second.

---

## 11. Show Cracked Passwords

```bash
hashcat --show -m <hash_mode> <hash_file>
```

**Example:**

```bash
hashcat --show -m 0 hashes.txt
```

Use this after the cracking attempt to retrieve recovered credentials cleanly.

---

## 12. Basic Hash Workflow

```bash
# Identify hash
hashid -m '<hash>'

# Dictionary attack
hashcat -a 0 -m <mode> hashes.txt /usr/share/wordlists/rockyou.txt

# Dictionary + rules
hashcat -a 0 -m <mode> hashes.txt \
/usr/share/wordlists/rockyou.txt \
-r /usr/share/hashcat/rules/best64.rule

# Mask attack
hashcat -a 3 -m <mode> hashes.txt '<mask>'

# Show recovered passwords
hashcat --show -m <mode> hashes.txt
```

---

## 13. Attack Decision

```
What do I know?
      │
      ├── common passwords?
      │       ↓
      │    Dictionary
      │
      ├── likely mutations?
      │       ↓
      │    Dictionary + Rules
      │
      ├── known structure?
      │       ↓
      │    Mask
      │
      └── very large unknown space?
              ↓
          broader search
```

The exact order depends on the target and available information.

---

## 14. Hashcat Mental Model

```
Hash
 ↓
Identify hash type
 ↓
Find Hashcat mode (-m)
 ↓
Choose attack strategy (-a)
 ↓
Generate candidates
 ↓
Hash candidates
 ↓
Compare
 ↓
Recovered?
 ↓
hashcat --show
```

---

## 15. Must-Remember Commands

```bash
# Help / hash modes
hashcat --help

# Identify hash + Hashcat mode
hashid -m '<hash>'

# Dictionary
hashcat -a 0 -m <mode> <hashes> <wordlist>

# Dictionary + rules
hashcat -a 0 -m <mode> <hashes> <wordlist> -r <ruleset>

# Mask
hashcat -a 3 -m <mode> <hashes> '<mask>'

# Show results
hashcat --show -m <mode> <hashes>

# Rules
ls -l /usr/share/hashcat/rules
```

---

## 16. The Two Numbers You Must Never Mix Up

```
-a = ATTACK
-m = HASH MODE
```

**Examples:**

```
-a 0 → dictionary
-a 3 → mask

-m 0    → MD5
-m 100  → SHA1
-m 1000 → NTLM
```

---

## 17. One-Line Memory Rule

> **Hashcat: `-m` tells it what the hash is; `-a` tells it how to attack it.**
