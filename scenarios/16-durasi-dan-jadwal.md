# Durasi & Jadwal Ideal Penetration Testing

> Estimasi untuk **1 tester**, aplikasi web skala menengah (10–50 endpoint).  
> Skala tim atau aplikasi kompleks: kalikan 1.5–2×.

---

## Ringkasan Durasi Semua Skenario

| # | Skenario | Black Box | Grey Box | White Box |
|---|----------|-----------|----------|-----------|
| 1 | Lokal (10.x.x.x) | 2 hari | 1.5 hari | 3–4 hari |
| 2 | Internal (210.57.x.x) | 3–4 hari | 2–3 hari | 5–7 hari |
| 3 | Internet Publik | 5–7 hari | 3–5 hari | 7–14 hari |

**Skenario tambahan (ditambahkan ke durasi di atas):**

| # | Skenario Tambahan | Tambahan Waktu |
|---|------------------|---------------|
| 10 | CMS (WordPress/Joomla) | +1–2 hari |
| 11 | SPA + REST/GraphQL API | +2–3 hari |
| 12 | E-Commerce + Payment | +3–4 hari |
| 13 | SSO / OAuth 2.0 | +2–3 hari |
| 14 | Mobile Backend (APK) | +3–5 hari |

---

## Faktor yang Mempengaruhi Durasi

| Faktor | Efek pada Durasi |
|--------|-----------------|
| Jumlah endpoint > 100 | +50–100% |
| Ada WAF/CDN | +1–2 hari |
| Scope termasuk subdomain | +1–3 hari |
| Perlu bypass certificate pinning (mobile) | +1 hari |
| Harus buat laporan formal | +1–2 hari |
| Tim 2 tester | −30% |
| Tim 3+ tester | −40–50% |

---

## Jadwal Detail per Skenario

---

### Skenario 1: Lokal Black Box — 2 Hari

```
HARI 1 (8 jam)
─────────────────────────────────────────────────────
08:00–09:00  Setup environment, buat folder struktur, inisiasi tools
09:00–10:30  Host discovery + port scan (nmap -sV -sC --top-ports 1000)
10:30–12:00  Web fingerprinting (whatweb, nikto, curl headers)
12:00–13:00  ISTIRAHAT
13:00–15:00  Directory/file enumeration (gobuster, ffuf)
15:00–17:00  Manual testing: SQLi, XSS pada semua input yang ditemukan
17:00–17:30  Review temuan hari 1, prioritaskan untuk hari 2

HARI 2 (8 jam)
─────────────────────────────────────────────────────
08:00–09:00  Full port scan (jika belum) + UDP scan top-100
09:00–11:00  Exploitation temuan dari hari 1 (SQLi dump, XSS PoC, LFI)
11:00–12:00  Post-exploitation (jika dapat akses): recon internal
12:00–13:00  ISTIRAHAT
13:00–15:30  Testing vulnerability lanjutan (IDOR, CSRF, file upload)
15:30–17:00  Cleanup, screenshot PoC, tulis draft laporan
17:00–17:30  Final review
```

---

### Skenario 2: Lokal Grey Box — 1.5 Hari (12 Jam)

```
HARI 1 (8 jam)
─────────────────────────────────────────────────────
08:00–08:30  Setup Burp Suite, konfigurasi browser proxy
08:30–09:30  Login, browse seluruh aplikasi, biarkan Burp record
09:30–10:30  Port scan paralel sambil browsing
10:30–12:00  Analisis Burp sitemap — identifikasi semua endpoint & parameter
12:00–13:00  ISTIRAHAT
13:00–15:00  IDOR testing pada semua resource ID yang ditemukan
15:00–17:00  Business logic testing, mass assignment, privilege escalation
17:00–17:30  Review & dokumentasi

HARI 2 (4 jam — setengah hari)
─────────────────────────────────────────────────────
08:00–09:30  JWT/session token analysis, auth bypass attempts
09:30–11:00  Exploitation temuan kritis + PoC documentation
11:00–12:00  Tulis laporan + rekomendasi remediation
```

---

### Skenario 3: Lokal White Box — 3–4 Hari

```
HARI 1 — Setup & Code Review (8 jam)
─────────────────────────────────────────────────────
08:00–09:00  Setup: clone repo, scan tools (semgrep, PHPCS, composer audit)
09:00–11:00  Automated static analysis (semgrep, PHPCS security)
11:00–12:00  Review hasil scan, buat prioritas
12:00–13:00  ISTIRAHAT
13:00–15:00  Manual code review: authentication & authorization logic
15:00–17:00  Manual code review: database queries & input handling
17:00–17:30  Dokumentasi temuan code review

HARI 2 — Database & Config Review (8 jam)
─────────────────────────────────────────────────────
08:00–10:00  Database schema & data review (PII, password hashing)
10:00–12:00  Configuration review (.env, web server, PHP settings)
12:00–13:00  ISTIRAHAT
13:00–14:00  File permission audit + git history secret scan
14:00–17:00  Dynamic testing berdasarkan temuan code review
17:00–17:30  Dokumentasi

HARI 3 — Exploitation & Laporan (8 jam)
─────────────────────────────────────────────────────
08:00–11:00  Exploitation semua temuan + PoC dengan screenshot
11:00–12:00  Post-exploitation (jika relevant)
12:00–13:00  ISTIRAHAT
13:00–17:00  Tulis laporan lengkap (findings + fix code + CVSS scores)
17:00–17:30  Review laporan
```

---

### Skenario 4: Internal Black Box (210.57.x.x) — 3–4 Hari

```
HARI 1 — OSINT & Discovery (8 jam)
─────────────────────────────────────────────────────
08:00–09:00  Setup + koneksi ke jaringan internal (VPN jika diperlukan)
09:00–11:00  Passive OSINT: WHOIS, Shodan, reverse DNS, crt.sh
11:00–12:00  Analisis OSINT findings
12:00–13:00  ISTIRAHAT
13:00–15:30  Slow port scan (nmap dengan delay, hindari IDS)
15:30–17:00  Web fingerprinting semua service yang ditemukan
17:00–17:30  Dokumentasi hari 1

HARI 2 — Enumeration (8 jam)
─────────────────────────────────────────────────────
08:00–10:00  Directory & file enumeration (rate-limited)
10:00–12:00  Authentication testing, default credentials
12:00–13:00  ISTIRAHAT
13:00–15:00  Manual web application testing (SQLi, XSS, IDOR)
15:00–17:00  Automated scanning (nuclei, nikto dengan delay)
17:00–17:30  Review temuan

HARI 3 — Exploitation (8 jam)
─────────────────────────────────────────────────────
08:00–10:00  Exploit temuan kritis dari hari 1-2
10:00–12:00  Post-exploitation jika berhasil
12:00–13:00  ISTIRAHAT
13:00–16:00  Testing lanjutan area yang belum dicakup
16:00–17:30  Dokumentasi + screenshot PoC

HARI 4 — Laporan (4 jam)
─────────────────────────────────────────────────────
08:00–12:00  Tulis laporan lengkap + rekomendasi
```

---

### Skenario 5: Internal Grey Box — 2–3 Hari

```
HARI 1 — Authenticated Discovery (8 jam)
─────────────────────────────────────────────────────
08:00–08:30  Setup Burp Suite + login dengan akun yang diberikan
08:30–10:00  Browse seluruh aplikasi sebagai user, Burp record traffic
10:00–11:00  API discovery (ffuf, cari dokumentasi Swagger)
11:00–12:00  Analisis semua endpoint & parameter dari Burp
12:00–13:00  ISTIRAHAT
13:00–15:30  IDOR testing (horizontal + vertical access control)
15:30–17:00  Business logic testing (konteks aplikasi HR/ERP Indonesia)
17:00–17:30  Dokumentasi

HARI 2 — Deep Testing (8 jam)
─────────────────────────────────────────────────────
08:00–09:30  JWT/session token analysis + manipulation
09:30–11:00  Mass assignment testing pada semua PUT/PATCH
11:00–12:00  Privilege escalation attempts
12:00–13:00  ISTIRAHAT
13:00–15:00  Injection testing (SQLi, XSS) pada semua input
15:00–17:00  Data exposure audit (PII yang bocor ke user biasa)
17:00–17:30  Dokumentasi

HARI 3 — Exploitation & Laporan (4–8 jam)
─────────────────────────────────────────────────────
08:00–10:00  Exploitation + PoC documentation
10:00–12:00  Tulis laporan
12:00–13:00  ISTIRAHAT (opsional jika 4 jam sudah cukup)
13:00–15:00  Final review + rekomendasi PII & regulasi Indonesia
```

---

### Skenario 6: Internal White Box — 5–7 Hari

```
HARI 1 — Setup & Server Audit
  08:00–10:00  SSH ke server, OS & patch level audit
  10:00–12:00  Network security review (firewall, exposed services)
  13:00–17:00  Dependency audit (composer, npm, pip)

HARI 2 — Static Code Analysis
  08:00–10:00  Run automated tools (semgrep, PHPCS, CodeQL)
  10:00–12:00  Review hasil scanner
  13:00–17:00  Manual code review (auth, SQL, file handling)

HARI 3 — Database & Config
  08:00–10:00  Database schema & data security audit
  10:00–12:00  Configuration files review
  13:00–15:00  Git history secret scanning
  15:00–17:00  Infrastructure review (Docker, deploy scripts)

HARI 4 — Dynamic Testing
  08:00–12:00  Dynamic testing berdasarkan temuan code review
  13:00–17:00  Exploitation + post-exploitation

HARI 5 — Laporan
  08:00–12:00  Tulis laporan teknis (file:baris referensi, fix code)
  13:00–15:00  Tulis executive summary + kaitkan regulasi Indonesia
  15:00–17:00  Final review

HARI 6–7 (jika sistem kompleks): Testing lanjutan & iterasi laporan
```

---

### Skenario 7: Internet Black Box — 5–7 Hari

```
HARI 1 — Passive OSINT (8 jam)
─────────────────────────────────────────────────────
08:00–09:00  Setup tools, verifikasi authorization letter
09:00–11:00  WHOIS, DNS, Shodan, Censys, crt.sh
11:00–12:00  Google dorking (sensitif files, admin panels, errors)
12:00–13:00  ISTIRAHAT
13:00–14:30  GitHub/GitLab secret scanning
14:30–16:00  Wayback Machine — endpoint dan file lama
16:00–17:00  Subdomain enumeration passive (subfinder, amass)
17:00–17:30  Kompilasi temuan OSINT

HARI 2 — Active Reconnaissance (8 jam)
─────────────────────────────────────────────────────
08:00–09:00  CDN/WAF detection (wafw00f), cari IP asli
09:00–11:00  Subdomain active enumeration (gobuster dns)
11:00–12:00  Httpx — validasi subdomain aktif
12:00–13:00  ISTIRAHAT
13:00–15:00  Port scan semua live subdomain
15:00–17:00  Web fingerprinting (whatweb, HTTP headers, teknologi)
17:00–17:30  Prioritaskan target

HARI 3 — Web Application Scanning (8 jam)
─────────────────────────────────────────────────────
08:00–10:00  Nuclei scan (rate-limited)
10:00–12:00  Directory enumeration (gobuster dengan random agent)
12:00–13:00  ISTIRAHAT
13:00–15:00  Nikto + manual checking
15:00–17:00  API endpoint discovery (swagger, js parsing)
17:00–17:30  Dokumentasi

HARI 4–5 — Manual Testing & Exploitation (16 jam)
─────────────────────────────────────────────────────
  Testing lengkap: SQLi, XSS, IDOR, CSRF, LFI, SSRF
  WAF bypass techniques jika diperlukan
  Exploitation temuan kritis + PoC

HARI 6–7 — Laporan (16 jam)
─────────────────────────────────────────────────────
  Executive summary + technical findings
  CVSS scoring + risk rating
  Remediation recommendations
  Final review + presentasi (jika diperlukan)
```

---

### Skenario 8: Internet Grey Box — 3–5 Hari

```
HARI 1 — Auth & API Discovery
  08:00–09:00  Login, capture token, setup Burp
  09:00–11:00  Browse semua fitur, API documentation hunting
  11:00–12:00  Extract endpoints dari JavaScript source
  13:00–15:00  API endpoint fuzzing (ffuf)
  15:00–17:00  Analisis JWT/OAuth tokens

HARI 2 — IDOR & Authorization
  08:00–12:00  IDOR testing sistematik (semua resource ID)
  13:00–15:00  Privilege escalation testing
  15:00–17:00  Mass assignment testing

HARI 3 — Business Logic & Deep Testing
  08:00–10:00  Business logic flaws
  10:00–12:00  Race condition testing
  13:00–15:00  Auth bypass, session testing
  15:00–17:00  CORS, DOM XSS, open redirect

HARI 4–5 — Exploitation + Laporan
  Exploitation + PoC documentation
  Laporan lengkap + bug bounty style reports
```

---

### Skenario 9: Internet White Box — 7–14 Hari

```
MINGGU 1 (5 hari)
  Hari 1: Git history scan + dependency audit
  Hari 2: Cloud security audit (AWS/GCP/Azure)
  Hari 3: CI/CD pipeline + IaC review
  Hari 4: Static code analysis
  Hari 5: Manual code review kritis

MINGGU 2 (5–9 hari)
  Hari 6–7: Dynamic testing berdasarkan code review
  Hari 8–9: Exploitation + post-exploitation
  Hari 10: Third-party integration testing
  Hari 11–14: Laporan komprehensif
```

---

## Jadwal Skenario Tambahan

### Skenario 10: CMS (WordPress) — Tambahan 1–2 Hari

```
HARI +1
  08:00–09:00  WPScan / Joomscan / Droopescan
  09:00–11:00  Plugin/theme vulnerability research (CVE)
  11:00–12:00  User enumeration (REST API, author archive)
  13:00–15:00  Exploitation (plugin vuln, brute force, XML-RPC)
  15:00–17:00  Post-access: webshell, persistence

HARI +2 (jika sistem besar)
  Manual plugin testing + laporan khusus CMS
```

---

### Skenario 11: SPA + API — Tambahan 2–3 Hari

```
HARI +1 — Discovery
  08:00–11:00  JS source analysis, endpoint extraction
  11:00–12:00  API documentation (Swagger, Postman)
  13:00–15:00  GraphQL introspection (jika ada)
  15:00–17:00  API endpoint fuzzing

HARI +2 — Testing
  08:00–12:00  OWASP API Top 10 testing
  13:00–15:00  JWT attacks
  15:00–17:00  CORS + DOM XSS + open redirect

HARI +3 — Exploitation + Laporan API
```

---

### Skenario 12: E-Commerce — Tambahan 3–4 Hari

```
HARI +1  Map seluruh purchase flow + price manipulation testing
HARI +2  Coupon/discount attacks + race condition testing
HARI +3  Payment callback testing + PCI-DSS audit
HARI +4  Account takeover + laporan bisnis impact
```

---

### Skenario 13: SSO/OAuth — Tambahan 2–3 Hari

```
HARI +1  OAuth flow reconnaissance + redirect_uri testing
HARI +2  Token manipulation + scope escalation + PKCE testing
HARI +3  Cross-app token replay + laporan (impact ke semua integrated apps)
```

---

### Skenario 14: Mobile Backend — Tambahan 3–5 Hari

```
HARI +1  APK decompile + static analysis (MobSF, jadx)
HARI +2  Certificate pinning bypass setup + traffic interception
HARI +3  API testing dari mobile context + hidden endpoint testing
HARI +4  Firebase/cloud backend testing + deep link abuse
HARI +5  Exploitation + laporan (APK findings + server findings)
```

---

## Estimasi Waktu Per Fase (Semua Skenario)

| Fase | % Waktu Total | Keterangan |
|------|--------------|------------|
| Reconnaissance | 20–30% | Lebih lama di Black Box & Internet |
| Scanning & Enumeration | 15–20% | Lebih cepat di White Box |
| Vulnerability Analysis | 20–30% | Lebih mendalam di White Box |
| Exploitation | 15–20% | Relatif sama semua skenario |
| Post-Exploitation | 5–10% | Lebih relevan di Black Box |
| **Reporting** | **20–25%** | **Sering diremehkan — jangan dikurangi** |

---

## Template Jadwal di Claude Code

**Prompt:**
```
Saya akan melakukan pentest selama [X hari] pada aplikasi [deskripsi]
menggunakan metode [black/grey/white box].
Akses: [lokal/internal/internet]
Tester: [1 orang / tim X orang]
Scope: [deskripsi scope]

Buatkan jadwal harian yang realistis dengan:
1. Alokasi waktu per aktivitas (format HH:MM–HH:MM)
2. Tools yang digunakan di setiap slot
3. Output/deliverable setiap akhir hari
4. Buffer time untuk temuan tak terduga
5. Checkpoint go/no-go untuk lanjut ke fase berikutnya
```
