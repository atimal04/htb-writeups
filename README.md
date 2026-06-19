# htb-writeups

Penetration test reports on retired HackTheBox machines.  
Each report follows a professional structure: executive summary, vulnerability analysis, technical reproduction, and remediation recommendations.

| Machine     | OS      | Difficulty | Key techniques                                                    |
|-------------|---------|------------|-------------------------------------------------------------------|
| Nibbles     | Linux   | Easy       | Web exploitation, sudo misconfiguration                           |
| Forest      | Windows | Easy       | AS-REP roasting, DCSync, Exchange privileges                      |
| Grandpa     | Windows | Easy       | IIS WebDAV, token impersonation                                   |
| Bounty      | Windows | Easy       | IIS misconfiguration, token impersonation                         |
| Sauna       | Windows | Easy       | AS-REP roasting, Winlogon cleartext, DCSync                       |
| Vault       | Linux   | Medium     | Pivoting, OpenVPN tunnel, GPG                                     |
| Pterodactyl | Linux   | Medium     | CVE-2025-49132, PAM bypass, XFS race condition                    |
| Garfield    | Windows | Hard       | RODC, KeyList attack, RBCD, scriptPath injection                  |
| Inject      | Linux   | Easy       | CVE-2022-22963 Spring RCE, credential reuse, Ansible cron abuse   |
