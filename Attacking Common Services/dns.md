# DNS Attacking — CPTS Cheatsheet

## 1. DNS Basics

**DNS = Domain Name System**

Converts:

```text
domain name → IP address

example.com → 93.184.216.34
```

### Ports

| Protocol | Port | Purpose                           |
| -------- | ---: | --------------------------------- |
| UDP      |   53 | Normal DNS queries                |
| TCP      |   53 | Zone transfers / larger responses |

**Important:** DNS has always supported both UDP and TCP. UDP is the normal transport, while TCP is particularly important for **AXFR zone transfers**.

---

# 2. DNS Enumeration

Start by identifying the DNS service and version.

```bash
nmap -Pn -p53 -sV -sC <TARGET>
```

Example:

```bash
nmap -Pn -p53 -sV -sC 10.10.110.213
```

Look for:

```text
53/tcp open domain ISC BIND ...
```

### Why version detection matters

Knowing the DNS implementation/version can help identify:

* Configuration weaknesses
* Known vulnerabilities
* Possible zone-transfer behavior
* Useful follow-up enumeration

---

# 3. DNS Record Types

Know these:

| Record  | Meaning                                  |
| ------- | ---------------------------------------- |
| `A`     | Domain → IPv4                            |
| `AAAA`  | Domain → IPv6                            |
| `CNAME` | Alias → another hostname                 |
| `MX`    | Mail server                              |
| `NS`    | Authoritative name server                |
| `SOA`   | Zone authority information               |
| `TXT`   | Arbitrary text / verification / SPF etc. |
| `PTR`   | IP → hostname                            |
| `SRV`   | Service location                         |

### Security relevance

```text
A       → identify hosts
AAAA    → identify IPv6 hosts
CNAME   → discover third-party dependencies
MX      → identify mail infrastructure
NS      → identify DNS servers
TXT     → potentially reveal useful information
SRV     → identify services
```

---

# 4. Basic DNS Queries with `dig`

### A record

```bash
dig <DOMAIN> A
```

### MX

```bash
dig <DOMAIN> MX
```

### NS

```bash
dig <DOMAIN> NS
```

### TXT

```bash
dig <DOMAIN> TXT
```

### CNAME

```bash
dig <DOMAIN> CNAME
```

### SOA

```bash
dig <DOMAIN> SOA
```

### Query a specific DNS server

```bash
dig @<DNS_SERVER> <DOMAIN>
```

Example:

```bash
dig @10.10.110.213 inlanefreight.htb
```

---

# 5. DNS Zone Transfer — AXFR

## What is it?

A **zone transfer** copies DNS zone information from one DNS server to another.

Normally:

```text
Primary DNS
     ↓
Secondary DNS
```

The problem occurs when the server incorrectly allows unauthorized clients to request the entire zone.

An attacker can potentially retrieve:

```text
subdomains
hostnames
IP addresses
mail servers
internal systems
TXT records
other DNS records
```

### Attack

```bash
dig AXFR @<DNS_SERVER> <DOMAIN>
```

Example:

```bash
dig AXFR @ns1.inlanefreight.htb inlanefreight.htb
```

Alternative:

```bash
dig axfr @10.129.110.213 inlanefreight.htb
```

### Successful result

You may see:

```text
admin.inlanefreight.htb
hr.inlanefreight.htb
support.inlanefreight.htb
...
```

That can reveal the organization's DNS namespace immediately.

---

# 6. Zone Transfer Mental Model

Normal DNS:

```text
Attacker
   │
   │ "What is the IP of www?"
   ↓
DNS Server
   │
   └──→ A record
```

AXFR:

```text
Attacker
   │
   │ "Give me the entire zone"
   ↓
Misconfigured DNS Server
   │
   └──→ Entire DNS zone
```

### Key point

**AXFR is not inherently a vulnerability.**

The vulnerability is **improper access control allowing unauthorized zone transfers**.

---

# 7. Fierce

`fierce` can perform DNS reconnaissance and look for zone-transfer opportunities.

```bash
fierce --domain <DOMAIN>
```

Example:

```bash
fierce --domain zonetransfer.me
```

Useful for discovering:

```text
NS
SOA
subdomains
DNS servers
zone-transfer information
```

---

# 8. Subdomain Enumeration

The goal is to discover:

```text
example.com
├── www.example.com
├── mail.example.com
├── vpn.example.com
├── dev.example.com
├── support.example.com
└── internal.example.com
```

Why?

Every discovered hostname potentially exposes:

* A new IP
* A new web application
* A different service
* A third-party provider
* A potential CNAME takeover

---

# 9. Subfinder

Passive subdomain enumeration:

```bash
subfinder -d <DOMAIN>
```

Example:

```bash
subfinder -d inlanefreight.com
```

Verbose:

```bash
subfinder -d inlanefreight.com -v
```

Concept:

```text
Subfinder
   ↓
Public sources
   ↓
DNS/subdomain information
   ↓
Candidate subdomains
```

---

# 10. Subbrute

Useful for DNS brute-forcing, particularly in internal environments.

Clone:

```bash
git clone https://github.com/TheRook/subbrute.git
```

Enter directory:

```bash
cd subbrute
```

Specify DNS resolver:

```bash
echo "ns1.inlanefreight.com" > ./resolvers.txt
```

Run:

```bash
./subbrute.py <DOMAIN> -s ./names.txt -r ./resolvers.txt
```

Concept:

```text
names.txt
   ↓
DNS queries
   ↓
Resolver
   ↓
Valid subdomains
```

### When Subbrute is useful

Especially when:

```text
Internal network
      +
Limited/no Internet access
      +
Known DNS resolver
```

---

# 11. CNAME Enumeration

Once you have subdomains, check their CNAME records.

```bash
host <SUBDOMAIN>
```

or:

```bash
dig <SUBDOMAIN> CNAME
```

Example:

```bash
host support.inlanefreight.com
```

Potential result:

```text
support.inlanefreight.com is an alias for inlanefreight.s3.amazonaws.com
```

Now you know:

```text
support.inlanefreight.com
        ↓
CNAME
        ↓
AWS S3
```

This is where **subdomain takeover investigation** begins.

---

# 12. Subdomain Takeover

## Core idea

A company creates:

```text
support.example.com
        ↓
CNAME
        ↓
third-party-service.example-provider.com
```

Later, the company stops using the third-party service but forgets to remove the CNAME.

If the external resource becomes claimable, an attacker may be able to register/claim it.

Result:

```text
support.example.com
        ↓
old CNAME
        ↓
attacker-controlled resource
```

---

# 13. Subdomain Takeover Workflow

```text
1. Enumerate subdomains
          ↓
2. Find CNAME records
          ↓
3. Identify third-party provider
          ↓
4. Determine whether resource still exists
          ↓
5. Check provider-specific takeover conditions
          ↓
6. Confirm whether resource can actually be claimed
```

### Example

```text
support.example.com
        ↓
CNAME
        ↓
example.s3.amazonaws.com
        ↓
NoSuchBucket
```

A `NoSuchBucket` response can be an **indicator**, but it is **not by itself proof of takeover**.

You must verify that:

1. The CNAME still exists.
2. The referenced resource is actually unclaimed.
3. The provider allows the resource to be claimed.
4. The claimed resource would actually serve the affected subdomain.

---

# 14. CNAME Takeover Mental Model

```text
Company DNS
     │
     │ CNAME
     ↓
Third-party resource
     │
     X  Resource deleted
     │
     ↓
Dangling DNS record
     │
     ↓
Potential takeover
     │
     ↓
Attacker-controlled resource
```

### Why it's dangerous

A trusted hostname such as:

```text
login.example.com
support.example.com
cdn.example.com
```

can potentially become attacker-controlled while still appearing to belong to the legitimate organization.

Possible impact can include:

* Phishing
* Cookie-related attacks depending on configuration
* CSRF-related abuse
* CORS-related abuse
* CSP bypass scenarios
* Brand/reputation abuse

Impact depends heavily on the application's configuration.

---

# 15. Takeover Reference

Useful reference:

```text
can-i-take-over-xyz
```

It documents provider-specific takeover conditions and known vulnerable/non-vulnerable services.

**Remember:** A dangling CNAME ≠ automatically exploitable takeover.

---

# 16. DNS Spoofing / Cache Poisoning

DNS spoofing means causing a victim to receive a **false DNS answer**.

Normal:

```text
victim
  ↓
DNS query: example.com
  ↓
DNS server
  ↓
1.2.3.4
```

Spoofed:

```text
victim
  ↓
DNS query
  ↓
attacker-controlled/fake response
  ↓
5.6.7.8
```

The victim is then directed toward the attacker's chosen destination.

---

# 17. Local DNS Spoofing

In a local network, DNS spoofing can be combined with **MITM**.

Example tool:

```text
Ettercap
```

Another option:

```text
Bettercap
```

### Ettercap DNS configuration

Edit:

```bash
sudo nano /etc/ettercap/etter.dns
```

Example:

```text
inlanefreight.com      A   192.168.225.110
*.inlanefreight.com    A   192.168.225.110
```

Concept:

```text
Victim asks:
inlanefreight.com?

        ↓

Attacker's spoofed DNS response

        ↓

192.168.225.110
```

---

# 18. Ettercap Attack Concept

High-level flow:

```text
Victim
  │
  │ DNS request
  ↓
MITM position
  │
  ↓
Fake DNS response
  │
  ↓
Attacker IP
  │
  ↓
Fake/malicious service
```

In an authorized internal lab, Ettercap can be configured with the victim as one target and the gateway as the other, then the `dns_spoof` plugin can provide the forged DNS responses.

---

# 19. DNS Attack Categories

Keep these three concepts separate:

### Zone Transfer

```text
Misconfigured DNS
       ↓
Unauthorized AXFR
       ↓
Entire zone disclosure
```

**Goal:** Information disclosure.

---

### Subdomain Takeover

```text
Dangling CNAME
       ↓
Third-party resource unavailable
       ↓
Resource potentially claimable
       ↓
Attacker controls subdomain
```

**Goal:** Control a trusted hostname.

---

### DNS Spoofing

```text
DNS response manipulation
       ↓
Victim receives false IP
       ↓
Traffic redirected
```

**Goal:** Redirect victims.

---

# 20. DNS Enumeration Workflow

Use this during CPTS-style labs:

```text
                 DNS
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
   Port/version         DNS records
   enumeration               │
        │              ┌─────┴─────┐
        ↓              ↓           ↓
      Nmap            NS          CNAME
                       │            │
                       ↓            ↓
                  Zone Transfer   Takeover
                       │
                       ↓
                    AXFR
                       │
                       ↓
               Hosts/subdomains
                       │
                       ↓
              Further enumeration
```

Practical workflow:

```bash
# 1. Find DNS
nmap -Pn -p53 -sV -sC <TARGET>

# 2. Identify nameservers
dig <DOMAIN> NS

# 3. Identify SOA
dig <DOMAIN> SOA

# 4. Try zone transfer
dig AXFR @<DNS_SERVER> <DOMAIN>

# 5. Passive subdomain enumeration
subfinder -d <DOMAIN>

# 6. Check discovered CNAMEs
host <SUBDOMAIN>

# 7. Query CNAME directly
dig <SUBDOMAIN> CNAME

# 8. Investigate potential dangling third-party resources
```

---

# 21. What to Remember for CPTS

| Technique                   | What you're looking for     | Main tool               |
| --------------------------- | --------------------------- | ----------------------- |
| DNS enumeration             | DNS service/version         | `nmap`                  |
| Record enumeration          | A/AAAA/MX/NS/TXT/CNAME/SOA  | `dig`                   |
| Zone transfer               | Misconfigured AXFR          | `dig AXFR`              |
| DNS reconnaissance          | Nameservers/subdomains      | `fierce`                |
| Passive subdomain discovery | Publicly known subdomains   | `subfinder`             |
| DNS brute forcing           | Hidden subdomains           | `subbrute`              |
| CNAME enumeration           | Third-party dependencies    | `host`, `dig`           |
| Subdomain takeover          | Dangling/claimable resource | CNAME + provider checks |
| DNS spoofing                | False DNS answers           | Ettercap/Bettercap      |

## Golden Rules

```text
53/UDP  → normal DNS
53/TCP  → especially important for AXFR

AXFR    → entire DNS zone

NS      → where authoritative DNS lives

CNAME   → alias / third-party dependency

Dangling CNAME
        ↓
Potential takeover
        ↓
Verify provider-specific exploitability

DNS spoofing
        ↓
Fake DNS answer
        ↓
Victim redirected
```

**Most important CPTS takeaway:** DNS is not just “resolve domain → IP.” It is an information source. Start with **NS/SOA → AXFR → subdomains → CNAMEs → third-party dependencies**, because each step can reveal the next attack surface.
