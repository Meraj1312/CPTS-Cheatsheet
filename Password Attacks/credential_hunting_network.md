# Credential Hunting in Network Traffic

## Core Concept

Credentials can sometimes be recovered directly from network traffic when protocols are unencrypted or improperly configured.

```text
Network traffic
      ↓
Protocol
      ↓
Authentication exchange
      ↓
Credential / hash / token
      ↓
Reuse / crack / authenticate
```

Four important questions:

```text
1. What protocol is being used?
2. Is the traffic encrypted?
3. Where does authentication occur?
4. What credential material can be recovered?
```

---

# Plaintext vs Protected Protocols

| Plaintext / weaker transport    | Protected counterpart                |
| ------------------------------- | ------------------------------------ |
| HTTP                            | HTTPS                                |
| FTP                             | FTPS / SFTP                          |
| POP3                            | POP3S                                |
| IMAP                            | IMAPS                                |
| SMTP                            | SMTP with TLS / SMTPS                |
| LDAP                            | LDAPS / LDAP with TLS                |
| SNMPv1/v2c                      | SNMPv3                               |
| DNS                             | DoH / DoT                            |
| RDP without protected transport | RDP with TLS/NLA                     |
| SMB without encryption          | SMB encryption / protected transport |

Important:

```text
SFTP ≠ FTP + TLS
SFTP = file transfer over SSH

FTPS = FTP protected by TLS
```

---

# Wireshark

Open a PCAP/PCAPNG:

```bash id="qa8tcq"
wireshark <capture.pcapng>
```

Useful display filters:

### IP

```text id="hc2i2q"
ip.addr == 10.10.10.10
```

### Source IP

```text id="w4ilnk"
ip.src == 10.10.10.10
```

### Destination IP

```text id="2b6fbb"
ip.dst == 10.10.10.20
```

### Port

```text id="oxg65k"
tcp.port == 80
```

### HTTP

```text id="y29pwe"
http
```

### DNS

```text id="1r9hb8"
dns
```

### ICMP

```text id="7byk2f"
icmp
```

### TCP SYN packets

```text id="6m2r4g"
tcp.flags.syn == 1 && tcp.flags.ack == 0
```

### HTTP POST

```text id="7agwjd"
http.request.method == "POST"
```

### HTTP containing a string

```text id="zks7vu"
http contains "passw"
```

Other useful searches:

```text
http contains "user"
http contains "login"
http contains "password"
http contains "token"
```

### TCP stream

```text id="1sn12a"
tcp.stream eq 53
```

Use this to isolate one TCP conversation.

Alternatively:

```text
Right-click packet
    ↓
Follow
    ↓
TCP Stream
```

### MAC address

```text id="a7yubr"
eth.addr == 00:11:22:33:44:55
```

---

# Finding HTTP Credentials

Start with:

```text id="ohdfxw"
http
```

Then:

```text id="xkrd4h"
http.request.method == "POST"
```

Look for request bodies such as:

```text
username=admin&password=Password123
```

or headers such as:

```text
Authorization: Basic ...
```

Remember:

```text
Base64 ≠ encryption
```

---

# FTP Credential Hunting

Plain FTP may expose authentication directly.

Look for:

```text id="fbzj9q"
USER
PASS
```

Wireshark:

```text id="laa7e2"
ftp
```

Follow the TCP stream to reconstruct the conversation.

---

# Pcredz

Purpose:

```text
Automatically search network traffic for credential material.
```

Run against a capture:

```bash id="6e1y8t"
./Pcredz -f demo.pcapng -t -v
```

Check installed version/help:

```bash id="d9x2a1"
./Pcredz -h
```

Potential findings include:

```text
FTP credentials
HTTP Basic credentials
HTTP form credentials
POP credentials
IMAP credentials
SMTP credentials
SNMP community strings
NTLM material
Kerberos authentication material
```

---

# Important Pcredz Findings

### FTP

```text
FTP User: ...
FTP Pass: ...
```

This can be plaintext.

### SNMP

```text
SNMPv2 Community string: ...
```

Treat as a network-management secret, not exactly a normal username/password.

### NTLM

Can reveal authentication material from:

```text
SMB
HTTP
LDAP
MSSQL
RPC
```

A captured NTLM exchange is not automatically a plaintext password.

### Kerberos

Pcredz can identify supported Kerberos authentication material such as:

```text
AS-REQ Pre-Authentication etype 23
```

This may become relevant for offline password attacks against AD credentials.

---

# Credential Types Found in Traffic

```text
Plaintext password
      ↓
Use directly where authorized

HTTP Basic credential
      ↓
Decode / recover credential

SNMP community string
      ↓
Test authorized SNMP access

NTLM authentication material
      ↓
Identify format
      ↓
Crack or use appropriate authentication technique

Kerberos authentication material
      ↓
Identify format
      ↓
Potential offline attack
```

---

# Network Credential Hunting Workflow

```text
PCAP / live capture
        ↓
Identify hosts
        ↓
Identify protocols
        ↓
Find unencrypted / weakly protected protocols
        ↓
Filter traffic
        ↓
Follow stream
        ↓
Inspect authentication
        ↓
Find password / hash / token
        ↓
Identify credential type
        ↓
Reuse or crack
```

---

# Key Filters to Memorize

```text
http
http.request.method == "POST"
http contains "passw"

ftp

dns

icmp

tcp.port == 80

tcp.stream eq 53

ip.addr == 10.10.10.10

ip.src == 10.10.10.10

ip.dst == 10.10.10.20
```

---

# Mental Model

```text
Host credential hunting
    ↓
Search files, history, memory, keyrings

Network credential hunting
    ↓
Search packets, protocols, streams

Credential dumping
    ↓
Extract from known credential stores

Credential cracking
    ↓
Recover plaintext from hash/material
```

## Golden Rules

```text
1. Plaintext protocols can expose credentials directly.
2. HTTPS/TLS protects application content in transit; it does not make traffic invisible.
3. Follow the TCP stream when individual packets are hard to understand.
4. HTTP POST requests are a high-value place to inspect for login data.
5. Base64 is encoding, not encryption.
6. NTLM/Kerberos traffic may expose authentication material rather than plaintext passwords.
7. Pcredz automates detection of many known credential patterns.
8. Always identify exactly what credential material you captured before deciding how to use or crack it.
9. Network credential hunting can turn one compromised host into access to another system through credential reuse or lateral movement.
```
