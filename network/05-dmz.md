# Network Pentest: DMZ (Demilitarized Zone)

**Target:** Server di zona DMZ — web server, mail server, DNS, reverse proxy  
**Tujuan:** Verifikasi DMZ benar-benar memisahkan internet dari jaringan internal  
**Durasi:** 3–5 hari  
**Risiko kritis:** Pivot dari DMZ yang dikompromikan → jaringan internal

---

## Arsitektur DMZ yang Ditest

```
Internet
    ↓
[Firewall Luar]
    ↓
[DMZ Zone]
  ├── Web Server (80/443)
  ├── Mail Server (25/587)
  ├── DNS Server (53)
  └── Reverse Proxy / WAF
    ↓
[Firewall Dalam]
    ↓
[Internal Network]
  ├── Database Server
  ├── AD/LDAP
  └── File Server
```

---

## Phase 1: DMZ Server Enumeration

```bash
# Scan semua host di DMZ (dari internet atau dari dalam DMZ)
nmap -sV -sC -p- dmz_subnet/24 \
  -oN scans/dmz_full_scan.txt

# Fokus pada service yang berhadapan dengan internet
nmap -sV -p 21,22,25,53,80,110,143,443,587,993,995,8080,8443 \
  dmz_server_ip

# SSL/TLS audit
sslyze dmz_server_ip:443
testssl.sh dmz_server_ip:443
```

**Prompt Claude Code:**
```
Hasil scan DMZ zone:

192.168.100.10 (Web/Reverse Proxy):
  80/tcp  nginx 1.18.0
  443/tcp nginx 1.18.0, TLS 1.0 diaktifkan (deprecated!)
  22/tcp  OpenSSH 7.4

192.168.100.20 (Mail Server):
  25/tcp  Postfix 3.3.0
  587/tcp STARTTLS
  110/tcp POP3 (plain text!)
  143/tcp IMAP (plain text!)

192.168.100.30 (DNS):
  53/udp  BIND 9.11.3 (outdated — CVE ada)
  53/tcp  BIND zone transfer enabled?

Identifikasi kerentanan dan prioritaskan attack vector.
```

---

## Phase 2: Pivot Testing dari DMZ ke Internal

Ini adalah test terpenting di DMZ — apakah dari DMZ bisa reach jaringan internal?

```bash
# Dari shell di DMZ server (setelah exploit web app misalnya):

# Test koneksi ke internal network
ping 10.0.1.10           # internal server
nc -zv 10.0.1.10 3306    # database
nc -zv 10.0.1.20 389     # LDAP/AD
nc -zv 10.0.1.20 445     # SMB

# Scan internal dari DMZ
nmap -sn 10.0.1.0/24
nmap -sV -p 22,80,443,445,3306,3389 10.0.1.0/24

# Jika ada akses: setup tunnel untuk akses dari attacker machine
# Port forwarding via SSH
ssh -L 3306:10.0.1.5:3306 user@dmz_server  # forward DB port ke local
```

**Prompt Claude Code:**
```
Berhasil RCE pada web server di DMZ (192.168.100.10).
Dari shell DMZ, test koneksi ke internal:

nc -zv 10.0.1.5 3306  → SUCCESS ← DMZ bisa reach database langsung!
nc -zv 10.0.1.20 445  → BLOCKED
nc -zv 10.0.1.20 389  → BLOCKED (LDAP)
ping 10.0.1.0/24      → beberapa host reply

Dari temuan ini:
1. Severity: database accessible dari DMZ — apa dampaknya?
2. Cara setup tunnel ke database melalui compromised DMZ server
3. Apakah bisa dump database dari posisi ini?
4. Attack chain lengkap: internet → web app vuln → DMZ RCE → pivot ke DB
5. Rekomendasi arsitektur yang benar (firewall rules antara DMZ dan internal)
```

---

## Phase 3: Mail Server Testing

```bash
# SMTP relay test — apakah bisa relay email ke mana saja?
telnet mail.target.co.id 25
EHLO attacker.com
MAIL FROM: <fake@attacker.com>
RCPT TO: <victim@external.com>   # apakah diterima?
DATA
Subject: Test relay
Test message
.
QUIT

# Email header injection
# User enumeration via SMTP VRFY/EXPN
nc mail.target.co.id 25
VRFY admin
VRFY root
EXPN users
```

**Prompt Claude Code:**
```
Mail server Postfix di DMZ (192.168.100.20):
- Open relay test: BERHASIL kirim ke external domain → open relay!
- VRFY command: enabled → user enumeration mungkin
- TLS: menggunakan STARTTLS tapi tidak enforce (MITM possible)
- POP3/IMAP tanpa SSL tersedia di port 110/143

Dampak dan rekomendasi:
1. Open relay: bagaimana dieksploitasi untuk spam/phishing?
2. VRFY: bagaimana enumerate valid email account?
3. Risiko credential sniffing via plain POP3/IMAP
4. Rekomendasi: disable open relay, enforce TLS, disable VRFY
```

---

## Phase 4: DNS Zone Transfer

```bash
# Test DNS zone transfer (AXFR) — mendapat seluruh DNS records
dig axfr target.co.id @192.168.100.30
nslookup
  server 192.168.100.30
  ls -d target.co.id

# Jika berhasil → mendapat semua hostname internal!
# Biasanya mengungkap: database.internal, admin.internal, vpn.internal, dsb
```

---

## Checklist DMZ Pentest

```
[ ] Enumerate semua server di DMZ
[ ] SSL/TLS audit setiap service
[ ] Test pivot: dari DMZ ke internal (coba semua port ke semua zona)
[ ] Mail server: open relay, VRFY, clear-text auth
[ ] DNS: zone transfer, cache poisoning
[ ] Web server: versi outdated, HTTP methods berbahaya (PUT/DELETE)
[ ] Test firewall rules: internet → DMZ, DMZ → internal
[ ] Laporan: diagram arsitektur dengan temuan per zona
```

---

## Durasi Estimasi

| Sub-skenario | Durasi |
|-------------|-----------|----------|
| DMZ service audit | 1–2 hari | 3–4 hari |
| Full DMZ pentest + pivot testing | 3–5 hari | 7–10 hari |
