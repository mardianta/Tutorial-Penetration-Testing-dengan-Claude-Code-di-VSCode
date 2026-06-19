# Skenario 7: Aplikasi Internet Publik — Black Box

**Akses:** Internet publik — siapapun bisa akses  
**Metode:** Black Box — hanya tahu domain/URL target  
**Contoh target:** `https://www.targetapp.co.id`  
**Perhatian:** Wajib ada **written authorization** sebelum memulai

---

## Perbedaan Utama vs Skenario Lokal/Internal

| Aspek | Lokal | Internal | Internet |
|-------|-------|----------|----------|
| OSINT scope | Tidak ada | Terbatas | **Penuh** |
| CDN/WAF | Jarang | Kadang | **Sering ada** |
| IP real server | Terlihat | Terlihat | Mungkin tersembunyi CDN |
| Subdomain | N/A | Terbatas | **Banyak** |
| Legal risk | Rendah | Sedang | **Paling tinggi** |
| Historical data | Tidak ada | Sedikit | **Wayback, Shodan, dll** |

---

## Phase 1: Passive OSINT (Tidak Menyentuh Target)

### Domain & IP Intelligence

```bash
# WHOIS
whois targetapp.co.id

# DNS Records
dig targetapp.co.id ANY
dig +short targetapp.co.id
dig +short www.targetapp.co.id
dig +short mail.targetapp.co.id  # MX → IP email server

# Cari IP asli di balik Cloudflare
# Tools: securitytrails.com, censys.io, shodan.io
# Query Shodan: ssl.cert.subject.cn:"targetapp.co.id"
```

### Subdomain Enumeration (Passive)

```bash
# Certificate Transparency Logs
curl -s "https://crt.sh/?q=%.targetapp.co.id&output=json" \
  | python3 -c "import sys,json; [print(e['name_value']) for e in json.load(sys.stdin)]" \
  | sort -u > recon/passive/subdomains_crt.txt

# Subfinder (passive OSINT)
subfinder -d targetapp.co.id -o recon/passive/subdomains_subfinder.txt

# Amass passive
amass enum -passive -d targetapp.co.id \
  -o recon/passive/subdomains_amass.txt
```

**Prompt Claude Code:**
```
Passive OSINT untuk targetapp.co.id (pentest yang diotorisasi).

Subdomains ditemukan dari crt.sh:
[LIST SUBDOMAIN]

WHOIS: [OUTPUT]
Shodan hasil: [DATA]

Analisis:
1. Subdomain mana yang paling menarik untuk diserang?
2. Apakah ada subdomain staging/dev/test yang mungkin kurang terlindungi?
3. Apakah ada IP yang tidak di-proxy CDN (IP asli server)?
4. Informasi organisasi apa yang bisa kita petakan dari OSINT ini?
5. Langkah OSINT lanjutan: Google dorks, LinkedIn, GitHub?
```

### Google Dorking

```
# Cari file sensitif
site:targetapp.co.id ext:sql | ext:bak | ext:env | ext:log
site:targetapp.co.id inurl:admin | inurl:login | inurl:dashboard

# Cari error messages (info disclosure)
site:targetapp.co.id "Warning: mysql_" | "Fatal error:" | "stack trace"
site:targetapp.co.id "Index of /"

# Cari di Wayback Machine
# https://web.archive.org/web/*/targetapp.co.id/*

# Cari di GitHub
"targetapp.co.id" site:github.com
"targetapp.co.id" "password" site:github.com
```

---

## Phase 2: Active Reconnaissance

### CDN/WAF Detection

```bash
# Deteksi Cloudflare
curl -I https://www.targetapp.co.id 2>/dev/null | grep -i "cf-ray\|cf-cache\|cloudflare"

# Deteksi WAF
wafw00f https://www.targetapp.co.id

# Alternatif: nmap WAF detection
nmap --script http-waf-detect https://www.targetapp.co.id
```

**Prompt Claude Code:**
```
Target menggunakan Cloudflare WAF (terdeteksi dari header CF-Ray).

Bagaimana cara:
1. Menemukan IP asli server di balik Cloudflare?
   (cek: historical DNS, subdomain yang bypass CDN, MX record, mail server)
2. Bypass Cloudflare untuk scanning langsung ke origin server?
3. Menyesuaikan payload agar tidak terblokir WAF saat testing?
4. Apakah ada risiko hukum jika bypass CDN pihak ketiga (Cloudflare)?
```

### Subdomain Active Enumeration

```bash
# Active DNS brute force
gobuster dns -d targetapp.co.id \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -o recon/active/subdomains_active.txt \
  -t 20

# Httpx — cek subdomain yang aktif
cat recon/passive/subdomains_crt.txt | httpx -o recon/active/live_subdomains.txt
```

---

## Phase 3: Web Application Scanning

```bash
# Untuk setiap subdomain aktif:
TARGET="https://www.targetapp.co.id"

# Fingerprint
whatweb "$TARGET" -v --log-json=scans/web/whatweb.json

# Directory scan dengan WAF evasion
gobuster dir \
  -u "$TARGET" \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -t 10 \
  --delay 500ms \
  --random-agent \
  -o scans/web/gobuster.txt

# Nuclei — CVE dan misconfiguration scan
nuclei -u "$TARGET" \
  -t nuclei-templates/ \
  -severity critical,high,medium \
  -o scans/web/nuclei.txt \
  -rate-limit 5

# Nikto dengan stealth options  
nikto -h "$TARGET" -Pause 2 -useragent "Mozilla/5.0 ..." \
  -o scans/web/nikto.txt
```

**Prompt Claude Code:**
```
Black box pentest pada https://www.targetapp.co.id (authorized).
Teknologi terdeteksi: WordPress 6.1.1, PHP 8.0, nginx

Nuclei findings:
[PASTE NUCLEI OUTPUT]

Gobuster menemukan:
/wp-admin, /wp-content/uploads/, /wp-json/

Prioritaskan attack vector dan berikan next steps untuk:
1. WordPress specific vulnerabilities
2. API endpoint /wp-json/ — enumerate users, posts sensitif
3. Plugin/theme vulnerabilities
4. File upload jika ada
```

---

## Phase 4: Exploitation dengan WAF Awareness

**Prompt Claude Code:**
```
SQLi testing pada https://targetapp.co.id/search?q=test
WAF: Cloudflare (terdeteksi)

Normal payload terblokir: ' OR 1=1--
Berikan:
1. WAF bypass techniques untuk SQLi
2. Encoded payloads yang sering lolos Cloudflare
3. Time-based blind SQLi yang lebih stealth
4. HTTP header injection sebagai alternatif
5. Cara menggunakan sqlmap dengan tamper scripts untuk bypass WAF
```

```bash
# SQLmap dengan WAF bypass
sqlmap -u "https://targetapp.co.id/search?q=1" \
  --tamper=space2comment,between,randomcase \
  --random-agent \
  --delay=2 \
  --level=3 \
  --risk=2 \
  --dbs \
  --batch
```

---

## Phase 5: Laporan untuk Klien Internet

Laporan untuk aplikasi internet harus menyertakan:

**Prompt Claude Code:**
```
Buatkan executive summary untuk klien (CISO level) dari black box pentest
https://www.targetapp.co.id:

Temuan:
- SQL Injection di /search endpoint — Critical
- WordPress 6.1.1 dengan 3 plugin outdated — High
- Admin panel /wp-admin accessible tanpa 2FA — Medium
- Sensitive file exposed: /backup/db.sql.gz — Critical
- Subdomain staging.targetapp.co.id dengan debug mode — High

Fokus pada:
1. Business impact (bukan teknis)
2. Risiko reputasi dan hukum
3. Potensi kerugian finansial
4. Timeline remediasi yang realistis
5. Quick wins yang bisa dilakukan dalam 48 jam
```

---

## Checklist Black Box Internet

```
[ ] Dapatkan written authorization (simpan sebagai bukti!)
[ ] Passive OSINT: WHOIS, DNS, crt.sh, Shodan
[ ] Google dorking
[ ] GitHub/GitLab secret scanning
[ ] Wayback Machine — temukan endpoint lama
[ ] Subdomain enumeration (passive + active)
[ ] CDN/WAF detection
[ ] Cari IP real di balik CDN
[ ] Fingerprint semua subdomain aktif
[ ] Nuclei scan (dengan rate limiting)
[ ] Directory enumeration (dengan random agent)
[ ] Manual testing: SQLi, XSS, IDOR dengan WAF bypass
[ ] Test authentication dan session
[ ] Dokumentasikan semua dengan timestamp (penting untuk legal)
[ ] Laporan executive + teknis
```

---

## Pertimbangan Legal & Etika

```
SEBELUM MEMULAI:
✓ Dapatkan surat otorisasi tertulis
✓ Tentukan scope dengan jelas (subdomain mana yang boleh ditest)
✓ Tentukan jam testing (jangan ganggu business hours)
✓ Siapkan emergency contact klien
✓ Verifikasi IP tester untuk whitelist di WAF klien

SELAMA TESTING:
✓ Jangan lakukan DoS/DDoS
✓ Jangan modifikasi/hapus data produksi
✓ Batasi rate request agar tidak ganggu availability
✓ Log semua aktivitas (timestamp, tool, target, result)
✓ Stop dan hubungi klien jika temukan data breach yang sudah ada

SETELAH TESTING:
✓ Hapus semua backdoor/webshell yang ditanam
✓ Hapus data yang di-dump dari server
✓ Laporkan temuan critical segera (tidak tunggu laporan final)
```
