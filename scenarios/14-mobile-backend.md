# Skenario 14 (Tambahan): Mobile Backend / API-First Application

**Berlaku untuk:** Aplikasi yang diakses via mobile app (Android/iOS)  
**Mengapa perlu tutorial khusus:** Reverse engineering APK membuka attack surface yang tidak terlihat dari web

---

## Kenapa Mobile Backend Berbeda

- **APK reverse engineering** → temukan hardcoded API keys, endpoint tersembunyi
- **Certificate pinning** → harus di-bypass untuk intercept traffic
- **API tanpa web frontend** → tidak ada HTML untuk dilihat, pure API
- **Debug/test endpoint** → sering tertinggal di production APK
- **Deep link abuse** → open redirect via intent URL

---

## Phase 1: APK Analysis

### Setup Android Testing Environment

```bash
# Tools yang diperlukan:
# - apktool (decompile APK)
# - jadx (decompile ke Java)
# - MobSF (Mobile Security Framework)
# - Genymotion / Android emulator
# - Frida (dynamic instrumentation)

# Download dan decompile APK
apktool d targetapp.apk -o decompiled/

# Decompile ke Java readable code
jadx -d jadx_output/ targetapp.apk

# Static analysis dengan MobSF
docker run -it --rm -p 8000:8000 opensecurity/mobile-security-framework-mobsf
# Upload APK ke http://localhost:8000
```

### Extract Secrets dari APK

```bash
# Cari hardcoded strings
grep -r "api_key\|apikey\|secret\|password\|token\|endpoint\|baseurl" \
  jadx_output/ --include="*.java" -l

# Cari URL endpoint tersembunyi
grep -roh "https\?://[^\"']*" jadx_output/ | sort -u > recon/passive/apk_endpoints.txt

# Cari Base64 encoded secrets
grep -r "BuildConfig\." jadx_output/ --include="*.java"
cat jadx_output/com/target/app/BuildConfig.java

# Cari secrets di resources
grep -r "string\|key\|secret" decompiled/res/ | grep -v "\.xml:"
cat decompiled/assets/config.json 2>/dev/null
```

**Prompt Claude Code:**
```
Decompiled Android APK dari target mobile app.
Ditemukan di BuildConfig.java:
- API_BASE_URL = "https://api.targetapp.co.id/v1"
- API_KEY = "sk-prod-xxxxxxxxxxxxxxxxxxx"
- ADMIN_ENDPOINT = "https://api.targetapp.co.id/internal/admin"
- DEBUG_MODE = false (tapi kita tahu ada debug endpoint)

Dan di strings.xml:
- firebase_api_key = "AIzaSyXXXXXXXX"
- maps_api_key = "AIzaSyYYYYYYYY"

Analisis:
1. API_KEY hardcoded — apa yang bisa dilakukan penyerang dengan ini?
2. ADMIN_ENDPOINT — test apakah bisa diakses tanpa auth admin
3. Firebase API key — apa risikonya? (database rules misconfiguration?)
4. Maps API key — apakah bisa digunakan untuk billing abuse?
5. Prioritas eksploitasi
```

---

## Phase 2: Certificate Pinning Bypass

Untuk intercept traffic mobile app, perlu bypass certificate pinning:

### Dengan Frida (Recommended)

```bash
# Install Frida
pip install frida-tools

# Di Android emulator (dengan root):
# Download frida-server untuk arch Android
# Push ke device
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell /data/local/tmp/frida-server &

# Bypass certificate pinning dengan script universal
frida -U -l ssl_pinning_bypass.js -f com.target.app --no-pause

# ssl_pinning_bypass.js (menggunakan frida-codeshare)
# https://codeshare.frida.re/@pcipolloni/universal-android-ssl-pinning-bypass-with-frida/
```

### Dengan Objection (Lebih Mudah)

```bash
# Objection = Frida wrapper yang lebih user friendly
pip install objection

# Patch APK untuk inject Frida (tanpa root)
objection patchapk --source targetapp.apk

# Install patched APK
adb install targetapp.objection.apk

# Jalankan dengan Burp proxy
objection -g com.target.app explore

# Di objection console:
android sslpinning disable
android proxy set 192.168.1.100 8080  # arahkan ke Burp
```

**Prompt Claude Code:**
```
Saya berhasil bypass certificate pinning pada mobile app target.
Sekarang semua traffic dari app ter-intercept di Burp Suite.

Request yang menarik:
1. POST /api/v1/auth/login: {"username":"user","password":"pass","device_id":"abc123"}
   Response: {"token": "eyJ...", "user_id": 1042, "role": "user", "device_token": "xyz"}

2. GET /api/v1/sync?user_id=1042&last_sync=1234567890
3. POST /api/internal/log: {"level":"debug","message":"user action","data":{...}}
4. GET /api/v1/users/1042/contacts  ← endpoint tidak ada di web app!

Dari temuan ini:
1. Endpoint /api/internal/log — apakah bisa inject malicious data?
2. /contacts endpoint yang tidak ada di web — test IDOR dengan ID berbeda
3. device_id parameter — apakah bisa dimanipulasi untuk bypass device binding?
4. last_sync parameter di /sync — apakah ada SQLi atau data exposure?
```

---

## Phase 3: API Testing dari Mobile Context

```python
# mobile_api_tester.py

import requests

API_BASE = "https://api.targetapp.co.id/v1"
API_KEY = "sk-prod-xxxxxxxxxxxxxxxxxxx"  # dari APK
USER_TOKEN = "<JWT dari login>"

HEADERS = {
    "Authorization": f"Bearer {USER_TOKEN}",
    "X-API-Key": API_KEY,
    "User-Agent": "TargetApp/2.1.0 (Android 13; Samsung Galaxy S21)",
    "X-Device-ID": "abc123def456",
    "Content-Type": "application/json"
}

def test_internal_endpoints():
    """Test endpoint 'internal' yang ditemukan di APK"""
    internal_endpoints = [
        "/internal/admin",
        "/internal/debug",
        "/internal/users/all",
        "/internal/config",
        "/internal/log",
    ]
    for ep in internal_endpoints:
        resp = requests.get(f"https://api.targetapp.co.id{ep}",
                          headers=HEADERS)
        print(f"[{resp.status_code}] {ep}: {resp.text[:100]}")

def test_version_downgrade():
    """Test apakah API v0 atau beta masih ada"""
    versions = ["/v0/", "/v1/", "/v2/", "/beta/", "/dev/", "/test/"]
    for ver in versions:
        resp = requests.get(f"https://api.targetapp.co.id{ver}users/me",
                          headers=HEADERS)
        if resp.status_code != 404:
            print(f"[!] API version {ver} exists: {resp.status_code}")

test_internal_endpoints()
test_version_downgrade()
```

---

## Phase 4: Firebase Security Testing

Firebase sering ter-misconfigured di mobile apps:

```bash
# Dari API key di APK: AIzaSyXXXXXXXX
FIREBASE_PROJECT="targetapp-prod"

# Test Realtime Database rules
curl "https://targetapp-prod-default-rtdb.firebaseio.com/.json"
# Jika bukan 401 → database bisa dibaca tanpa auth!

# Test Firestore
curl "https://firestore.googleapis.com/v1/projects/targetapp-prod/databases/(default)/documents/users"
# Unauthorized harusnya 403

# Test Storage
curl "https://firebasestorage.googleapis.com/v0/b/targetapp-prod.appspot.com/o"
# List files?
```

**Prompt Claude Code:**
```
Firebase testing dari mobile app targetapp:

Realtime Database endpoint memberikan response 200 (bukan 401!):
curl "https://targetapp-prod-default-rtdb.firebaseio.com/.json"
Response: {"users": {"user_1042": {"name":"John","email":"john@t.com","phone":"08123"}}}

Firestore rules (ditemukan di decompiled code):
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if true;  ← SEMUA ORANG BISA READ/WRITE!
    }
  }
}

Analisis:
1. Data apa yang bisa diakses? (dump seluruh users collection?)
2. Apakah bisa WRITE ke database? (modifikasi data, hapus user)
3. Estimasi jumlah user yang terpapar
4. Cara PoC dump data via Firebase REST API
5. Fix: Firebase security rules yang benar
```

---

## Phase 5: Deep Link Testing

```bash
# Test Android deep links / intent URL
# Bisa melalui: browser → app intent
# adb shell am start -W -a android.intent.action.VIEW -d "targetapp://profile?user_id=1"

# Test open redirect via deep link
adb shell am start -W -a android.intent.action.VIEW \
  -d "targetapp://redirect?url=https://evil.com"

# Test XSS via deep link (jika ada WebView)
adb shell am start -W -a android.intent.action.VIEW \
  -d "targetapp://search?q=<script>alert(1)</script>"
```

---

## Checklist Mobile Backend Testing

```
[ ] Decompile APK (apktool + jadx)
[ ] Extract hardcoded secrets, endpoints, API keys
[ ] Static analysis dengan MobSF
[ ] Bypass certificate pinning (Frida/Objection)
[ ] Intercept semua API traffic dengan Burp
[ ] Test endpoint tersembunyi dari APK
[ ] Test API versioning (v0, beta, dev)
[ ] Firebase security rules testing
[ ] Deep link abuse testing
[ ] Test device_id / device binding bypass
[ ] Test token validity di web vs mobile API
[ ] IDOR pada endpoint mobile-specific
[ ] Laporan: sertakan APK findings + API findings
```

---

## Rekomendasi Keamanan Mobile Backend

**Prompt Claude Code:**
```
Buatkan rekomendasi security untuk tim developer mobile app:

Temuan:
1. API key hardcoded di APK
2. Certificate pinning tidak diimplementasikan
3. Firebase database world-readable
4. Debug endpoint aktif di production
5. Device ID mudah dipalsukan

Untuk setiap temuan:
1. Cara benar implementasi (dengan code snippet)
2. Prioritas perbaikan (immediate/high/medium)
3. Library/tool yang direkomendasikan
4. Cara test bahwa fix sudah benar

Bonus: panduan secure mobile API design (autentikasi, authorization, rate limiting)
```
