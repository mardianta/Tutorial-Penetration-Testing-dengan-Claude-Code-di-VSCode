# Skenario 4: Jaringan Internal (210.57.X.X) — Black Box

**Akses:** Jaringan internal organisasi / intranet (IP publik terbatas)  
**Metode:** Black Box — hanya IP/domain target, tanpa informasi lain  
**Contoh target:** `http://210.57.10.25` atau `http://apps.internal.org`  
**Catatan:** IP `210.57.x.x` adalah blok milik Indonesia (sering dipakai universitas/BUMN)

---

## Perbedaan dengan Skenario Lokal

| Aspek | Lokal (10.x.x.x) | Internal (210.57.x.x) |
|-------|-----------------|----------------------|
| OSINT publik | Tidak tersedia | Sebagian tersedia (WHOIS, Shodan) |
| Akses tester | Langsung di LAN | Perlu VPN atau akses dari dalam |
| Firewall | Minimal | Ada network firewall, mungkin IDS/IPS |
| DNS resolusi | Internal DNS | Mungkin ada di DNS publik |
| Scope Shodan | Tidak ada data | Mungkin ada historical data |

---

## Phase 1: OSINT untuk IP Internal

Meskipun internal, IP publik Indonesia sering punya data di sumber terbuka:

```bash
# WHOIS — cari info organisasi
whois 210.57.10.25

# Shodan — cek apakah pernah ter-expose
# Buka di browser: https://www.shodan.io/host/210.57.10.25

# Reverse DNS
nslookup 210.57.10.25
dig -x 210.57.10.25

# Traceroute — peta jaringan
traceroute 210.57.10.25
```

**Prompt Claude Code:**
```
OSINT untuk target internal 210.57.10.25:

WHOIS output: [OUTPUT]
Reverse DNS: [OUTPUT]
Shodan data (jika ada): [DATA]

Dari informasi ini:
1. Organisasi apa yang memiliki IP ini?
2. Subdomain atau host lain yang mungkin terhubung?
3. Port/service apa yang terlihat di Shodan?
4. Langkah OSINT selanjutnya yang relevan
```

---

## Phase 2: Network Scanning dengan Pertimbangan IDS/IPS

Karena ada kemungkinan IDS/IPS, lakukan scanning dengan lebih hati-hati:

```bash
# Scan perlahan untuk hindari trigger IDS
nmap -sV --top-ports 1000 210.57.10.25 \
  --scan-delay 1s \
  --max-rate 100 \
  -oN scans/nmap/initial_slow.txt

# Alternatif: scan dari beberapa port source
nmap -sV -p 80,443,8080,8443,22,21,25,3306 210.57.10.25 \
  -oN scans/nmap/common_ports.txt

# SYN scan (lebih stealth, butuh root)
sudo nmap -sS -T2 210.57.10.25 -oN scans/nmap/syn_scan.txt

# UDP scan untuk service tersembunyi
sudo nmap -sU --top-ports 100 210.57.10.25 \
  -oN scans/nmap/udp_scan.txt
```

**Prompt Claude Code:**
```
Saya melakukan black box pentest pada 210.57.10.25 (jaringan internal organisasi).
Target kemungkinan memiliki IDS/IPS.

Buatkan scanning strategy yang:
1. Meminimalkan chance terdeteksi IDS
2. Tetap mendapatkan informasi maksimal
3. Urutan: discovery → port scan → service enum → web scan
4. Tambahkan flag nmap yang tepat untuk setiap tahap
5. Berapa delay yang direkomendasikan?
```

---

## Phase 3: Web Application Testing

```bash
# Web fingerprinting
whatweb http://210.57.10.25 -v --log-brief=scans/web/whatweb.txt

# Directory enumeration — lebih hati-hati
gobuster dir \
  -u http://210.57.10.25 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -t 10 \
  --delay 200ms \
  -o scans/web/gobuster.txt

# Nikto dengan delay
nikto -h http://210.57.10.25 -Tuning 123bde -o scans/web/nikto.txt

# Check common paths untuk Indonesian enterprise apps
curl -s http://210.57.10.25/login
curl -s http://210.57.10.25/admin
curl -s http://210.57.10.25/api/v1/
curl -s http://210.57.10.25/swagger/
curl -s http://210.57.10.25/actuator/  # Spring Boot
```

**Prompt Claude Code — Konteks Aplikasi Indonesia:**
```
Target 210.57.10.25 adalah aplikasi internal organisasi Indonesia.
Dari fingerprinting: [HASIL WHATWEB/HEADER]

Berikan checklist testing khusus untuk:
1. Aplikasi enterprise Indonesia yang umum (SIMRS, SIMPEG, SIM-X)
2. Framework yang sering dipakai (CodeIgniter, Laravel, Spring Boot)
3. Default credentials yang sering ditemukan di lingkungan ini
4. Path/endpoint khas yang sering ada tapi tidak terlindungi
```

---

## Phase 4: Authentication Testing

```bash
# Test default credentials umum
hydra -L wordlists/users.txt -P wordlists/passwords.txt \
  http-post-form \
  "210.57.10.25/login:username=^USER^&password=^PASS^:Invalid" \
  -t 3 -w 3

# Wordlist default credentials Indonesia
cat > wordlists/indo_defaults.txt << 'EOF'
admin:admin
admin:admin123
admin:password
admin:123456
administrator:administrator
user:user
guest:guest
operator:operator
EOF
```

**Prompt Claude Code:**
```
Black box testing pada aplikasi internal 210.57.10.25.
Halaman login ditemukan di /login.

Buatkan:
1. Daftar default credentials yang umum untuk aplikasi enterprise Indonesia
2. Script Python untuk brute force dengan rate limiting ketat (1 req/2 detik)
3. Cara deteksi apakah ada account lockout policy
4. Cara bypass captcha yang mungkin ada di form login
```

---

## Phase 5: Network-Level Attacks (Jika dalam Jaringan Sama)

```bash
# ARP poisoning untuk MITM (jika tester di jaringan sama)
# PERHATIAN: Ini invasif, konfirmasi scope terlebih dahulu

# Passive sniffing dulu
sudo tcpdump -i eth0 host 210.57.10.25 -w loot/traffic.pcap

# Analisis traffic
tshark -r loot/traffic.pcap -T fields -e http.host -e http.request.uri \
  | head -50
```

---

## Pertimbangan Khusus Jaringan Internal

**Prompt Claude Code:**
```
Saya melakukan black box pentest pada aplikasi di 210.57.10.25.
Ini adalah jaringan internal organisasi (universitas/BUMN).

Pertimbangan khusus:
1. Apakah ada teknik OSINT spesifik untuk organisasi Indonesia?
2. Bagaimana cara mendeteksi apakah ada monitoring/SOC aktif?
3. Apa tanda-tanda IDS/IPS aktif dan bagaimana menyesuaikan pendekatan?
4. Jika ditemukan sensitive data (data mahasiswa/pegawai), protokol apa yang harus diikuti?
5. Bagaimana mendokumentasikan temuan agar tidak melanggar regulasi data Indonesia?
```

---

## Checklist Black Box Internal

```
[ ] WHOIS dan reverse DNS lookup
[ ] Cek Shodan/Censys untuk data historis
[ ] Slow port scan (hindari trigger IDS)
[ ] Web application fingerprinting
[ ] Directory enumeration (rate-limited)
[ ] Test default credentials
[ ] Manual testing: SQLi, XSS, IDOR
[ ] Check API endpoints dan swagger docs
[ ] Test authentication bypass
[ ] Dokumentasikan temuan + dampak ke organisasi
[ ] Laporkan segera jika temukan data sensitif (data breach protocol)
```
