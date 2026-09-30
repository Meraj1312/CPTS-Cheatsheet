# Pivoting, Tunneling & Port Forwarding 

## 1. Core Concept

Port forwarding redirects a connection from one port to another, usually tunneled over SSH (or SOCKS), so you can reach services that aren't directly routable from your attack host. Main use: pivoting through a compromised host into a network you can't reach directly.

```
Attack Host  ──SSH──▶  Pivot Host (dual-homed)  ──▶  Internal Network / internal-only service
```

**Three flavors:**

| Type              | Flag | Direction                                   | Use case                                  |
| ----------------- | ---- | -------------------------------------------- | ------------------------------------------ |
| Local forward      | `-L` | Local port → remote port                     | Reach a service only listening on the pivot |
| Dynamic (SOCKS)     | `-D` | Local SOCKS proxy → whole remote network      | Scan/access an entire internal subnet       |
| Remote (reverse)   | `-R` | Remote port → local port                      | Pivot host-only target calls back to you    |

---

## 2. Local Port Forwarding (`-L`)

Forward a port on the pivot host (often bound only to `localhost` there) to a local port on your attack box.

```
ssh -L <LOCAL_PORT>:localhost:<REMOTE_PORT> <USER>@<PIVOT_IP>
```

Example — MySQL on the pivot is bound to `127.0.0.1:3306`, forward it to your local `1234`:

```
ssh -L 1234:localhost:3306 ubuntu@10.129.202.64
```

Forward multiple services in one command:

```
ssh -L 1234:localhost:3306 -L 8080:localhost:80 ubuntu@10.129.202.64
```

### Verify the forward worked

```
netstat -antp | grep <LOCAL_PORT>
nmap -v -sV -p<LOCAL_PORT> localhost
```

**Use when:** you already know exactly which service/port on the pivot you want, and just need to reach it locally (e.g. to point a local exploit/tool at it).

---

## 3. Dynamic Port Forwarding / SOCKS Tunneling (`-D`)

Turns your SSH client into a SOCKS proxy so **any tool**, via `proxychains`, can route arbitrary traffic through the pivot into networks you have no direct route to.

```
ssh -D <LOCAL_PORT> <USER>@<PIVOT_IP>
```

Example:

```
ssh -D 9050 ubuntu@10.129.202.64
```

### Configure proxychains

Edit `/etc/proxychains.conf`, make sure the last line matches your listener:

```
socks4  127.0.0.1 9050
```

### Scan through the tunnel

```
proxychains nmap -v -sn 172.16.5.1-200          # host discovery
proxychains nmap -v -Pn -sT 172.16.5.19          # full TCP connect scan of one host
```

**Key limitations:**
- Only **full TCP connect scans** (`-sT`) work through proxychains — it can't interpret partial/half-open packets, so SYN scans give wrong results.
- ICMP (`ping`) often won't traverse the tunnel and Windows Defender blocks ICMP anyway — always scan with `-Pn` (skip host discovery) against Windows targets.
- Full-range scans over proxychains are slow; scan individual hosts/small ranges you already suspect are alive.

### Reach an RDP (or any) target through the tunnel

Once you've confirmed a port (e.g. 3389) is open, connect directly with a normal client through proxychains — no exploitation framework required:

```
proxychains xfreerdp /v:<TARGET_IP> /u:<USER> /p:'<PASSWORD>'
```

Other tools work the same way — `proxychains crackmapexec smb 172.16.5.0/23`, `proxychains impacket-mssqlclient ...`, `proxychains smbclient ...`, etc. — just prefix any TCP-based tool with `proxychains`.

**Use when:** you need to reach/scan more than one host or service on a network you can't route to directly.

---

## 4. Remote / Reverse Port Forwarding (`-R`)

Used when the **target itself can't reach your attack host** at all (no route out of its subnet), but it *can* reach the pivot. The pivot listens on a port and forwards anything that hits it back to a listener on your attack host.

```
ssh -R <PIVOT_INTERNAL_IP>:<PIVOT_LISTEN_PORT>:<YOUR_LISTEN_IP>:<YOUR_LISTEN_PORT> <USER>@<PIVOT_IP> -vN
```

- `-v` verbose (confirm the forward is actually set up)
- `-N` don't open a login shell, just forward

### Typical flow

1. Stand up a listener on your attack host for whatever callback you're expecting (reverse shell, HTTP callback, etc.) on `<YOUR_LISTEN_PORT>`.
2. Get your payload/agent onto the internal target (e.g. via a file transfer through the pivot — `scp` the payload to the pivot, then serve it with `python3 -m http.server` and pull it down on the target with `Invoke-WebRequest` or a browser).
3. Open the reverse forward from the pivot host:

   ```
   ssh -R 172.16.5.129:8080:0.0.0.0:8000 ubuntu@<PIVOT_IP> -vN
   ```

   This tells the pivot to listen on `172.16.5.129:8080` (its internal-facing IP) and relay anything received there to `0.0.0.0:8000` on your attack host.
4. Trigger the payload on the target — it connects to `<pivot_internal_ip>:8080`, which SSH forwards over the existing SSH channel back to your listener.
5. Confirm on the pivot's SSH debug output that a `forwarded-tcpip` channel opened, and check your listener for the incoming connection.

**Note:** the connection your listener sees will show as `127.0.0.1` — because it arrives over the local SSH socket, not the true origin IP. Use `netstat` on the pivot to confirm the real source if you need to.

**Use when:** the target is only reachable *outbound* toward the pivot (common with strict egress filtering, or a target with no route at all to your network).

**Non-Metasploit alternative for step 1–2:** you don't need `msfconsole`/`msfvenom` for this — any reverse-shell payload (a raw `nc`/PowerShell one-liner, a custom binary, Impacket-generated shellcode, etc.) and any listener (`nc -lvnp <port>`, or a plain `socat`) will work through the same `-R` tunnel. The forwarding mechanics are identical regardless of what generates the payload or catches the shell.

---

## 5. SSH Script for Ping Sweep

Pure-Python, SSH/pivot-friendly host + common-port sweep — no `nmap` or Metasploit binary required on the pivot box (handy when neither is installed there, or you don't want to drop extra tooling). Run it directly on the pivot over your SSH session, or through a forwarded/tunneled shell:

```bash
python3 -c "
import socket, concurrent.futures as f

ports = [22, 80, 445, 3389]

def chk(ip):
    open_ports = []
    for p in ports:
        try:
            s = socket.create_connection((ip, p), timeout=1)
            s.close()
            open_ports.append(p)
        except:
            pass
    if open_ports:
        print(ip, open_ports)

ips = [f'172.16.{o}.{h}' for o in (5, 6) for h in range(1, 255)]
with f.ThreadPoolExecutor(150) as e:
    list(e.map(chk, ips))
"
```
## Powershell script for Ping Sweep
```powershell
$ports = 22,80,445,3389

foreach ($subnet in 5,6) {
    foreach ($hostnum in 1..254) {
        $ip = "172.16.$subnet.$hostnum"

        foreach ($port in $ports) {
            try {
                $client = New-Object System.Net.Sockets.TcpClient
                $result = $client.BeginConnect($ip, $port, $null, $null)

                if ($result.AsyncWaitHandle.WaitOne(500) -and $client.Connected) {
                    Write-Host "$ip`t$port"
                }

                $client.Close()
            }
            catch {
                # Ignore connection failures
            }
        }
    }
}
```
**How it works:**
- Sweeps a set of `/24`s (edit the `o in (5, 6)` tuple and the `172.16.` prefix for your target range) across a small, common-service port list (`22, 80, 445, 3389` — edit as needed).
- Uses `socket.create_connection` (a real TCP connect, 1s timeout) per port — equivalent to `nmap -sT`, just handwritten.
- `ThreadPoolExecutor(150)` fans the checks out across 150 threads so a full sweep of a couple of /24s finishes quickly.
- Only prints IPs that had at least one open port — quiet by default, easy to grep/pipe.

**Use when:** you're on a pivot with only Python available, want a fast alive/port-sweep without touching disk (single one-liner, nothing to drop as a file), or want to avoid running `nmap`/Metasploit on a box where that might be logged or isn't installed.

---

## 6. Attack Chain Summary

```
Compromise dual-homed pivot host
        │
        ├── Know the exact port you want?  ──▶  -L local forward  ──▶  direct tool access
        │
        ├── Need to scan/reach a whole subnet? ──▶ -D + proxychains ──▶ nmap -sT / any tool prefixed
        │                                                                 with proxychains
        │
        └── Target can't route to you at all? ──▶ -R reverse forward ──▶ payload calls back
                                                                            through the pivot
                                                                            to your listener

        (no nmap/Metasploit on the pivot?) ──▶ SSH ping-sweep one-liner (Section 5)
```

---

## 7. Core Things to Remember

- `-L` = pull a remote port to you. `-D` = open a SOCKS tunnel to the whole network. `-R` = push your listener's reachability to the target's side.
- `proxychains` only reliably does full TCP connect scans (`-sT`) — never SYN scans — and needs `/etc/proxychains.conf` pointed at your `-D` port.
- Use `-Pn` when scanning Windows hosts through a tunnel — ICMP is commonly blocked and won't route through SOCKS cleanly anyway.
- `-R` connections on your listener will appear to originate from `127.0.0.1` (they arrive over the local SSH socket) — that's expected, not an error.
- You do not need Metasploit for any of this — `proxychains` + normal client tools (`xfreerdp`, `impacket-*`, `crackmapexec`, `nc`) covers scanning, service access, and catching reverse shells through any of the three forward types.
- Keep the SSH ping-sweep script handy for pivots where `nmap`/Metasploit aren't installed or you'd rather not drop them.
