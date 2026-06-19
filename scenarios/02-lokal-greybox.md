# Skenario 2: Aplikasi Lokal (10.X.X.X) — Grey Box

**Akses:** Jaringan lokal / LAN / VM internal  
**Metode:** Grey Box — informasi terbatas: 1 akun user, sebagian dokumentasi  
**Contoh target:** `http://10.10.10.5`

---

## Informasi Awal yang Tersedia

| Item | Status |
|------|--------|
| URL / IP target | ✓ Diberikan |
| Kredensial user biasa | ✓ 1 akun (misal: `user:password123`) |
| Source code | ✗ Tidak ada |
| Dokumentasi teknis | ✓ Sebagian (API docs, user manual) |
| Teknologi stack | ✓ Sebagian (misal: "pakai PHP dan MySQL") |
| Struktur database | ✗ Tidak diketahui |

---

## Keunggulan Grey Box vs Black Box

Grey box memungkinkan Anda:
- **Bypass fase enumeration awal** — langsung masuk ke aplikasi sebagai user
- **Mapping fitur lebih cepat** — tahu konteks bisnis dari dokumentasi
- **Test authorization issues** — IDOR, privilege escalation, akses horizontal/vertikal
- **Intercept authenticated requests** — API calls dengan session valid

---

## Phase 1: Reconnaissance Terarah

```bash
# Quick port scan — lebih efisien karena sudah tahu target
nmap -sV -sC --top-ports 1000 10.10.10.5 \
  -oN scans/nmap/initial_scan.txt

# Login dengan kredensial yang diberikan untuk mapping
# Gunakan browser + Burp Suite proxy
```

**Prompt Claude Code:**
```
Grey box pentest pada 10.10.10.5. Saya punya:
- Akun: user123 / pass123
- Dokumentasi: aplikasi manajemen karyawan dengan modul: login, profil, laporan, admin

Buatkan checklist testing terarah untuk:
1. Endpoint yang harus di-map setelah login
2. Parameter yang rawan IDOR (ID karyawan, ID laporan, dsb)
3. Fungsi yang harus dicoba untuk privilege escalation ke admin
4. Data sensitif yang mungkin bocor ke user biasa
```

---

## Phase 2: Authenticated Mapping dengan Burp Suite

### Setup Burp Suite sebagai Proxy

```
Browser → Proxy 127.0.0.1:8080 → Burp Suite → Target 10.10.10.5
```

```bash
# Start Burp Suite, set proxy di browser
# Login dengan akun yang diberikan
# Browse SEMUA fitur yang tersedia sebagai user
# Burp akan merekam semua request

# Export sitemap dari Burp:
# Target → Site map → klik kanan → Copy URLs in this host
```

**Prompt Claude Code — Analisis Burp Sitemap:**
```
Ini adalah daftar endpoint yang ditemukan Burp Suite setelah login sebagai user biasa
pada aplikasi di 10.10.10.5:

[PASTE LIST ENDPOINT DAN PARAMETER]

Identifikasi:
1. Endpoint yang mungkin mengandung IDOR (ada ID numerik/UUID?)
2. Endpoint yang seharusnya hanya untuk admin
3. Parameter yang mungkin rentan injection
4. API calls yang membawa data sensitif
Prioritaskan dari risiko tertinggi.
```

---

## Phase 3: Authorization Testing (IDOR & Privilege Escalation)

### IDOR Testing

```python
# vulnerabilities/idor_tester.py
import requests

BASE_URL = "http://10.10.10.5"
SESSION_COOKIE = "PHPSESSID=<your-session>"
HEADERS = {"Cookie": SESSION_COOKIE}

def test_idor(endpoint_template, id_range=range(1, 50)):
    """Test IDOR pada endpoint dengan parameter ID"""
    findings = []
    for obj_id in id_range:
        url = BASE_URL + endpoint_template.format(id=obj_id)
        resp = requests.get(url, headers=HEADERS)
        if resp.status_code == 200 and len(resp.text) > 100:
            findings.append({
                "id": obj_id,
                "url": url,
                "status": resp.status_code,
                "size": len(resp.text)
            })
            print(f"[+] Accessible: {url} ({len(resp.text)} bytes)")
    return findings

# Test berbagai endpoint
endpoints = [
    "/api/user/{id}/profile",
    "/api/report/{id}",
    "/download?file_id={id}",
    "/employee/{id}",
]

for ep in endpoints:
    print(f"\n[*] Testing IDOR: {ep}")
    test_idor(ep)
```

**Prompt Claude Code:**
```
Buatkan Python script IDOR tester yang lebih lengkap untuk aplikasi grey box.
Fitur:
1. Baca list endpoint dari file JSON (endpoint + parameter yang dicurigai IDOR)
2. Test dengan session user biasa (cookie dari Burp)
3. Bandingkan response — apakah data milik user lain bisa diakses?
4. Test juga dengan method PUT/DELETE untuk test IDOR write access
5. Generate report JSON dengan semua temuan
Target: http://10.10.10.5
```

### Horizontal & Vertical Privilege Escalation

```bash
# Test akses ke endpoint admin sebagai user biasa
curl -b "PHPSESSID=<session>" http://10.10.10.5/admin/
curl -b "PHPSESSID=<session>" http://10.10.10.5/admin/users
curl -b "PHPSESSID=<session>" http://10.10.10.5/api/admin/all-users

# Test manipulasi parameter role
# Intercept request di Burp, ubah: role=user → role=admin
# atau: user_type=0 → user_type=1
```

---

## Phase 4: Business Logic Testing

Karena tahu konteks bisnis dari dokumentasi, test logika yang tidak biasa:

**Prompt Claude Code:**
```
Aplikasi manajemen karyawan di 10.10.10.5. Dokumentasi menunjukkan:
- User dapat mengajukan cuti (form: tanggal, jenis, durasi)
- Manager dapat approve/reject
- Laporan absensi hanya bisa dilihat HR

Buatkan test case untuk business logic vulnerabilities:
1. Dapatkah user approve cuti sendiri? (parameter manipulation)
2. Dapatkah user mengajukan cuti dengan tanggal retroaktif?
3. Dapatkah user akses laporan absensi orang lain?
4. Apakah ada rate limiting pada pengajuan cuti?
5. Negative testing: apa yang terjadi jika input durasi = -1 atau 99999?
```

---

## Phase 5: Session & Authentication Testing

```bash
# Analisis JWT jika digunakan
# Decode JWT di jwt.io atau dengan Python:
python3 -c "
import base64, json
token = '<JWT_TOKEN>'
parts = token.split('.')
header = json.loads(base64.b64decode(parts[0] + '=='))
payload = json.loads(base64.b64decode(parts[1] + '=='))
print('Header:', header)
print('Payload:', payload)
"

# Test JWT none algorithm attack
# Test JWT weak secret bruteforce
```

**Prompt Claude Code:**
```
JWT token dari aplikasi grey box: [TOKEN]

Analisis:
1. Algoritma yang digunakan — apakah rentan?
2. Klaim di payload — adakah 'role', 'admin', 'user_id' yang bisa dimanipulasi?
3. Test 'none algorithm' attack
4. Jika menggunakan HS256, buatkan script untuk bruteforce secret dengan wordlist
5. Cara forge token admin jika secret ditemukan
```

---

## Checklist Grey Box Lokal

```
[ ] Login dengan akun yang diberikan
[ ] Map seluruh endpoint dengan Burp Suite
[ ] Identifikasi semua parameter ID (potensi IDOR)
[ ] Test IDOR horizontal (akses data user lain)
[ ] Test IDOR vertikal (akses fitur admin)
[ ] Analisis JWT/session token
[ ] Test business logic vulnerabilities
[ ] Test input validation pada semua form
[ ] Test file upload jika ada
[ ] Coba SQLi, XSS pada endpoint yang ditemukan
[ ] Dokumentasi semua temuan + PoC
[ ] Draft laporan dengan CVSS rating
```

---

## Keunggulan Pendekatan Grey Box

| Aspek | Black Box | Grey Box |
|-------|-----------|----------|
| Waktu discovery | Lama | Cepat (sudah punya akun) |
| Coverage | Terbatas ke public endpoint | Semua endpoint authenticated |
| IDOR testing | Susah tanpa session | Sangat efektif |
| Business logic | Hanya guess | Terarah dari dokumentasi |
| False negative | Tinggi | Lebih rendah |
