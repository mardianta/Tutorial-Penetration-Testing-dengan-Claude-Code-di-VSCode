# Skenario 6: Jaringan Internal (210.57.X.X) — White Box

**Akses:** Jaringan internal organisasi + akses penuh ke sistem  
**Metode:** White Box — source code, database, SSH ke server, semua akun  
**Contoh target:** `http://210.57.10.25` + SSH `root@210.57.10.25`

---

## Informasi Awal yang Tersedia

| Item | Status |
|------|--------|
| Source code | ✓ Full repository |
| Database | ✓ Schema + dump |
| Akses SSH/server | ✓ `root` atau `sudo` |
| Semua kredensial | ✓ Admin, DB, service accounts |
| Konfigurasi server | ✓ nginx/Apache, PHP-FPM |
| Konfigurasi jaringan | ✓ Firewall rules, network topology |
| Dokumentasi arsitektur | ✓ Lengkap |

---

## Fokus Khusus White Box Internal

Berbeda dengan white box lokal, white box jaringan internal menambahkan:
1. **Keamanan jaringan** — firewall rules, network segmentation
2. **Server hardening** — apakah server sudah di-harden?
3. **Dependency & patch level** — library/framework sudah up-to-date?
4. **Infrastructure as Code** — apakah ada konfigurasi Ansible/Terraform yang insecure?
5. **Log & monitoring** — apakah ada SIEM/monitoring aktif?

---

## Phase 1: Server Security Audit

```bash
# SSH ke server
ssh root@210.57.10.25

# Cek OS dan patch level
uname -a
cat /etc/os-release
apt list --upgradable 2>/dev/null | head -20  # Debian/Ubuntu
yum check-update 2>/dev/null | head -20       # CentOS/RHEL

# Cek user dan privilege
cat /etc/passwd | grep -v nologin
cat /etc/sudoers
id www-data  # user web server

# Cek service yang berjalan
systemctl list-units --type=service --state=active
netstat -tlnp
ss -tlnp

# Cek file SUID berbahaya
find / -perm -4000 -type f 2>/dev/null
```

**Prompt Claude Code:**
```
White box pentest pada server 210.57.10.25.
Output audit server:

uname -a: [OUTPUT]
Packages yang butuh update: [LIST]
SUID files: [LIST]
Active services: [LIST]
sudoers: [CONTENT]

Identifikasi:
1. Kernel version — apakah ada local privilege escalation exploit?
2. Packages yang rentan (CVE dengan patch tersedia)
3. SUID binary yang tidak standar dan berbahaya
4. Service yang tidak perlu berjalan (attack surface reduction)
5. Misconfiguration sudoers yang bisa dieksploitasi
```

---

## Phase 2: Dependency & Library Audit

```bash
# PHP dependencies
cd /var/www/html
composer audit  # cek vulnerabilities di packages

# Node.js (jika ada)
npm audit --json > vulnerabilities/npm_audit.json

# Python (jika ada)
pip-audit -o vulnerabilities/pip_audit.json

# Cek versi framework
php artisan --version        # Laravel
python -c "import django; print(django.__version__)"
```

**Prompt Claude Code:**
```
Output composer audit dari aplikasi Laravel di 210.57.10.25:
[PASTE AUDIT OUTPUT]

Analisis:
1. CVE mana yang paling critical untuk di-exploit?
2. Apakah ada vulnerability yang langsung exploitable via web?
3. Buatkan PoC untuk vulnerability library yang paling berbahaya
4. Urutan prioritas patching
```

---

## Phase 3: Network Security Configuration Review

```bash
# Cek firewall rules
iptables -L -n -v
ufw status verbose  # Ubuntu

# Cek apakah ada port internal yang tidak seharusnya exposed
netstat -tlnp | grep -v 127.0.0.1

# Cek database akses dari luar
mysql -h 210.57.10.25 -u root -p  # apakah MySQL bisa diakses dari luar?

# Cek Redis/Memcached tanpa auth
redis-cli -h 210.57.10.25 ping

# Cek apakah ada service yang bind ke 0.0.0.0
ss -tlnp | grep "0.0.0.0"
```

**Prompt Claude Code:**
```
Network configuration dari server 210.57.10.25:

iptables rules: [OUTPUT]
Listening services (ss -tlnp): [OUTPUT]
MySQL bind-address (my.cnf): [OUTPUT]

Identifikasi:
1. Service yang seharusnya tidak accessible dari jaringan
2. Apakah database exposed ke network? Risiko apa?
3. Apakah ada service tanpa autentikasi (Redis, Elasticsearch)?
4. Network segmentation — apakah server bisa reach server lain yang sensitif?
5. Rekomendasi firewall rules yang benar
```

---

## Phase 4: Complete Source Code Audit

```bash
# Setup static analysis tools
cd /var/www/html

# PHP: PHPCS Security Audit
composer require --dev pheromone/phpcs-security-audit
./vendor/bin/phpcs --standard=Security . --extensions=php \
  --report=json > vulnerabilities/phpcs_security.json

# PHP: RIPS / Psalm static analysis
composer require --dev vimeo/psalm
./vendor/bin/psalm --output-format=json > vulnerabilities/psalm.json

# Semgrep — multi-language
semgrep --config=p/security-audit . \
  --json --output vulnerabilities/semgrep.json
```

**Prompt Claude Code:**
```
Output static analysis dari kodebase Laravel (210.57.10.25):

PHPCS Security findings: [OUTPUT / JSON]
Semgrep findings: [OUTPUT]

Untuk setiap temuan:
1. Apakah ini true positive atau false positive?
2. Severity sebenarnya berdasarkan konteks kode
3. PoC exploit jika exploitable
4. Lokasi tepat: file + baris
5. Fix code yang direkomendasikan
```

---

## Phase 5: Database Security Audit

```sql
-- Koneksi ke database dengan credentials yang diberikan
mysql -u root -p app_database

-- 1. Cek password hashing
SELECT username, password, LENGTH(password) as hash_len 
FROM users LIMIT 10;
-- MD5 = 32 char, SHA1 = 40 char, bcrypt = 60 char

-- 2. Cek privilege user database
SELECT user, host, plugin FROM mysql.user;
SHOW GRANTS FOR 'app_user'@'localhost';

-- 3. Cek data sensitif tidak terenkripsi
SELECT column_name, table_name 
FROM information_schema.columns
WHERE column_name REGEXP 'nik|ktp|salary|gaji|password|token|secret'
AND table_schema = 'app_database';

-- 4. Cek audit log
SHOW VARIABLES LIKE 'general_log%';
SHOW VARIABLES LIKE 'log_bin%';

-- 5. Cek apakah ada data PII
SELECT COUNT(*), MIN(nik), MAX(nik) FROM employees;  -- apakah NIK disimpan plain?
```

**Prompt Claude Code:**
```
Database audit dari 210.57.10.25:

Password storage sample:
username=admin, password=5f4dcc3b5aa765d61d8327deb882cf99  (32 chars = MD5)

MySQL user privileges:
app_user: SELECT, INSERT, UPDATE, DELETE, FILE, SUPER

PII columns ditemukan: nik, no_hp, alamat, gaji (semua plain text)

General log: OFF
Binary log: OFF

Analisis dampak dan rekomendasi:
1. Risiko password MD5 tanpa salt
2. Bahaya privilege FILE dan SUPER untuk app_user
3. Risiko PII tidak terenkripsi (kaitkan dengan UU PDP Indonesia)
4. Rekomendasi perbaikan database security
```

---

## Phase 6: Infrastructure & Deployment Review

```bash
# Cek .env dan konfigurasi sensitif
cat /var/www/html/.env
cat /var/www/html/config/database.php

# Cek apakah debug mode aktif
grep "APP_DEBUG" /var/www/html/.env
grep "APP_ENV" /var/www/html/.env

# Cek permission file
ls -la /var/www/html/
ls -la /var/www/html/.env
ls -la /var/www/html/storage/

# Cek git history — mungkin ada credential yang pernah di-commit
git -C /var/www/html log --all --oneline
git -C /var/www/html log -p -- .env | grep "^+" | grep -i "password\|secret\|key"
```

**Prompt Claude Code:**
```
Infrastructure findings dari white box pentest 210.57.10.25:

.env file:
APP_ENV=production
APP_DEBUG=true          <-- INI MASALAH
DB_PASSWORD=Admin123!
JWT_SECRET=secret

File permissions:
.env: -rw-rw-rw- (world writable!)
storage/: drwxrwxrwx (world writable!)

Git history:
Found DB_PASSWORD=OldPassword di commit 3 bulan lalu

Buat laporan temuan infrastructure dengan severity dan rekomendasi.
```

---

## Checklist White Box Internal

```
[ ] SSH server audit (OS, patch level, users)
[ ] Dependency/library vulnerability scan (composer audit, npm audit)
[ ] Static code analysis (semgrep, PHPCS-Security)
[ ] Manual code review: auth, SQL queries, file handling
[ ] Database security audit (password hashing, privileges, PII)
[ ] Network security review (firewall, exposed services)
[ ] Configuration audit (.env, web server config, PHP settings)
[ ] Git history review (credential leaks)
[ ] File permission audit
[ ] Log & monitoring capability review
[ ] Dynamic testing berbasis temuan code review
[ ] Laporan dengan referensi file:baris dan fix code
[ ] Kaitkan temuan dengan regulasi (UU PDP, ISO 27001)
```

---

## Referensi Regulasi Indonesia

| Regulasi | Relevansi |
|----------|-----------|
| **UU PDP No. 27/2022** | Perlindungan data pribadi (NIK, data karyawan) |
| **Permenkominfo 4/2016** | Standar teknis sistem elektronik pemerintah |
| **ISO 27001** | Standar SMKI yang banyak diadopsi BUMN/instansi |
| **NIST SP 800-53** | Framework keamanan federal (referensi best practice) |

**Prompt Claude Code:**
```
Saya sedang menulis laporan white box pentest untuk instansi pemerintah Indonesia.
Temuan meliputi: data PII tidak terenkripsi, password MD5, debug mode aktif.

Kaitkan setiap temuan dengan:
1. UU PDP No. 27 Tahun 2022 — pasal yang relevan
2. Potensi sanksi jika terjadi breach
3. Rekomendasi teknis yang sesuai standar pemerintah Indonesia
4. Timeline remediasi yang realistis
```
