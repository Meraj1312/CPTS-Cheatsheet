# Custom Wordlists & Rules — CPTS Cheatsheet

**Core idea:** Build likely base words from target information, then use rules to generate predictable password variations.

---

## 1. Mental Model

```
OSINT / Target Information
          ↓
     Base words
          ↓
      Wordlist
          ↓
       Rules
          ↓
  Mutated candidates
          ↓
 Hashcat / John cracking
```

---

## 2. Common Human Password Patterns

```
Password
Password123
Password2026
Password09
Password!
P@ssword
P@ssw0rd
P@ssw0rd!
```

**Common additions:**

- Uppercase first letter
- Numbers
- Year
- Month
- Special character
- Leetspeak substitutions

---

## 3. Wordlist vs Rules

```
WORDLIST
= base words/candidates

RULE
= transformation applied to each base word
```

**Example:**

```
password
   ↓
c
   ↓
Password
```

**Another:**

```
password
   ↓
so0
   ↓
passw0rd
```

**Another:**

```
password
   ↓
c so0
   ↓
Passw0rd
```

---

## 4. Basic Hashcat Rule Functions

| Rule | Meaning | Example |
| --- | --- | --- |
| : | Do nothing | password → password |
| l | Lowercase all | PASSWORD → password |
| u | Uppercase all | password → PASSWORD |
| c | Capitalize first, lowercase rest | password → Password |
| sXY | Replace X with Y | so0: password → passw0rd |
| $X | Append character X | $!: password → password! |

---

## 5. Substitution

**Syntax:**

```
sXY
```

**Meaning:**

```
replace every X with Y
```

**Examples:**

```
so0
```

means:

```
o → 0
```

```
sa@
```

means:

```
a → @
```

**Example:**

```
password
   ↓ so0
passw0rd
```

```
password
   ↓ sa@
p@ssword
```

---

## 6. Rule Chaining

Rules are executed from left to right.

**Example:**

```
c so0
```

```
password
   ↓ c
Password
   ↓ so0
Passw0rd
```

**Example:**

```
c sa@ so0
```

```
password
   ↓ c
Password
   ↓ sa@
P@ssword
   ↓ so0
P@ssw0rd
```

---

## 7. Custom Rule File

**Create:**

```bash
nano custom.rule
```

**Example:**

```
:
c
so0
c so0
sa@
c sa@
c sa@ so0
$!
$! c
$! so0
$! sa@
$! c so0
$! c sa@
$! so0 sa@
$! c so0 sa@
```

Each line = one separate rule.

---

## 8. Generate Mutated Wordlist

**Base wordlist:**

```
password
```

**Run:**

```bash
hashcat --force password.list \
-r custom.rule \
--stdout
```

**Important:**

```
--stdout = print generated candidates
```

It does **NOT** crack a hash.

---

## 9. Save Mutated Wordlist

```bash
hashcat --force password.list \
-r custom.rule \
--stdout | sort -u > mut_password.list
```

**Pipeline:**

```
password.list
     ↓
custom.rule
     ↓
generated candidates
     ↓
sort
     ↓
remove duplicates
     ↓
mut_password.list
```

---

## 10. Why sort -u?

```
sort -u
```

means:

```
sort entries
+
remove duplicates
```

Useful when different rules generate the same password.

---

## 11. Pre-Built Rules

**Common Hashcat rules location:**

```
/usr/share/hashcat/rules
```

**View:**

```bash
ls -l /usr/share/hashcat/rules
```

**Common rulesets:**

```
best64.rule
dive.rule
leetspeak.rule
rockyou-30000.rule
```

---

## 12. Apply a Ruleset During Cracking

```bash
hashcat -a 0 -m <hashmode> \
<hashes> \
<wordlist> \
-r /usr/share/hashcat/rules/best64.rule
```

**Example:**

```bash
hashcat -a 0 -m 0 hashes.txt \
/usr/share/wordlists/rockyou.txt \
-r /usr/share/hashcat/rules/best64.rule
```

**Mental model:**

```
rockyou
   ↓
best64
   ↓
many password mutations
   ↓
Hashcat
```

---

## 13. Generate Candidates Without Cracking

```bash
hashcat <wordlist> \
-r <ruleset> \
--stdout
```

**Example:**

```bash
hashcat password.list \
-r custom.rule \
--stdout
```

**Save:**

```bash
hashcat password.list \
-r custom.rule \
--stdout | sort -u > custom_wordlist.txt
```

---

## 14. --force

```
--force
```

Can bypass certain Hashcat warnings/checks.

**Don't automatically use it every time.**

Understand warnings before suppressing them.

---

## 15. CeWL

CeWL creates custom wordlists by crawling websites and extracting words.

**General syntax:**

```bash
cewl <URL> [options]
```

**Example:**

```bash
cewl https://www.inlanefreight.com \
-d 4 \
-m 6 \
--lowercase \
-w inlane.wordlist
```

---

## 16. CeWL Options

| Option | Meaning |
| --- | --- |
| -d <n> | Spider/crawl depth |
| -m <n> | Minimum word length |
| --lowercase | Store extracted words in lowercase |
| -w <file> | Write results to file |

**Example:**

```
-d 4
```

→ crawl to depth 4.

```
-m 6
```

→ keep words with minimum length 6.

```
--lowercase
```

→ normalize extracted words to lowercase.

```
-w inlane.wordlist
```

→ save results.

---

## 17. Count Wordlist Entries

```bash
wc -l inlane.wordlist
```

**Example:**

```
326
```

means approximately 326 lines/entries.

---

## 18. CeWL → Rules

**Workflow:**

```
Website
   ↓
CeWL
   ↓
company-specific words
   ↓
Hashcat rules
   ↓
password candidates
```

**Example:**

```bash
cewl https://target.example \
-d 4 \
-m 6 \
--lowercase \
-w target.wordlist
```

**Then:**

```bash
hashcat target.wordlist \
-r custom.rule \
--stdout | sort -u > target_mutated.txt
```

---

## 19. Custom Wordlist Strategy

**Think about:**

- Company name
- Company products
- Locations
- Industries
- Technologies
- Sports
- Hobbies/interests
- Project names
- Other publicly exposed terminology

**Then create:**

```
base wordlist
```

and apply predictable transformations.

---

## 20. Targeted Password Generation

```
Known information
       ↓
Interesting keywords
       ↓
Base wordlist
       ↓
Password policy
       ↓
Custom rules
       ↓
Targeted candidate list
```

**Example:**

```
company
   ↓
company1
Company1
company2026
Company2026!
c0mpany2026!
```

---

## 21. Important Distinction

**Wordlist generation**

```
words → candidates
```

**Password cracking**

```
hash + candidates → password
```

**Rules**

```
candidate → transformed candidates
```

They are different stages.

---

## 22. Complete Workflow

```
1. Recon / OSINT
        ↓
2. Identify useful keywords
        ↓
3. Manual list or CeWL
        ↓
4. Base wordlist
        ↓
5. Apply custom/prebuilt rules
        ↓
6. Remove duplicates
        ↓
7. Use against target hashes
        ↓
8. Analyze recovered credentials
```

---

## 23. Must-Remember Commands

**Create custom rule**

```bash
nano custom.rule
```

**Generate candidates**

```bash
hashcat words.txt -r custom.rule --stdout
```

**Generate unique wordlist**

```bash
hashcat words.txt -r custom.rule --stdout | sort -u > mutated.txt
```

**List Hashcat rules**

```bash
ls -l /usr/share/hashcat/rules
```

**Use prebuilt rules**

```bash
hashcat -a 0 -m <mode> hashes.txt words.txt \
-r /usr/share/hashcat/rules/best64.rule
```

**Generate website wordlist**

```bash
cewl https://target.example \
-d 4 \
-m 6 \
--lowercase \
-w target.wordlist
```

**Count entries**

```bash
wc -l target.wordlist
```

---

## 24. Memory Block

```
CUSTOM WORDLISTS & RULES

WORDLIST
→ base words

RULE
→ transformation

--stdout
→ generate candidates instead of cracking

sort -u
→ sort + remove duplicates

CeWL
→ scrape useful words from a website

CeWL:
-d → depth
-m → minimum word length
--lowercase → lowercase output
-w → output file

Hashcat rules:
:       → do nothing
l       → lowercase
u       → uppercase
c       → capitalize
sXY     → replace X with Y
$X      → append X

-a 0    → dictionary attack
-r      → ruleset
```

---

## 25. One-Line Mental Model

> **Recon gives you the words, rules give you the human password patterns, and Hashcat turns those into candidates you can test against the hashes.**
