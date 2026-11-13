# htb-writeups

Writeups on retired HackTheBox machines.
Published for personal reference and interview purposes. Active machines live in the private repository until they are retired.

| Machine | OS | Difficulty | Key Techniques |
|---|---|---|---|
| Garfield | Windows | Hard | scriptPath/ACL abuse, Ligolo-ng pivot, RBCD, RODC Golden Ticket, DCSync |
| Abducted | Linux | Medium | Samba print injection, rclone abuse, symlink abuse |
| Forest | Windows | Medium | AS-REP Roasting, ACL chain (Exchange → WriteDACL), DCSync |
| Pterodactyl | Linux | Medium | Multi-level pivoting, PHP upload filter bypass, OpenVPN config RCE |
| Vault | Linux | Medium | Multi-stage pivoting, firewall source-port bypass, GPG-encrypted root flag |
| Bounty | Windows | Easy | web.config ASP upload, JuicyPotato (SeImpersonatePrivilege) |
| Grandpa | Windows | Easy | CVE-2017-7269 (WebDAV PROPFIND), Token Kidnapping |
| Inject | Linux | Easy | CVE-2022-22963 (Spring SpEL RCE), Ansible playbook privesc |
| Nibbles | Linux | Easy | Nibbleblog CVE-2015-6967 upload, password-less sudo script |
| Sauna | Windows | Easy | AS-REP Roasting, Active Directory, DCSync |
