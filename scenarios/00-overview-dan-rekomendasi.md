# Overview Skenario Penetration Testing

## Matriks Skenario × Metode

| # | Skenario Akses | Black Box | Grey Box | White Box |
|---|---------------|-----------|----------|-----------|
| 1 | **Lokal** — `10.X.X.X` (LAN/VM) | [01](01-lokal-blackbox.md) | [02](02-lokal-greybox.md) | [03](03-lokal-whitebox.md) |
| 2 | **Internal Terbatas** — `210.57.X.X` (intranet/VPN) | [04](04-internal-blackbox.md) | [05](05-internal-greybox.md) | [06](06-internal-whitebox.md) |
| 3 | **Internet Publik** — domain/IP publik | [07](07-internet-blackbox.md) | [08](08-internet-greybox.md) | [09](09-internet-whitebox.md) |

---

## Definisi Metode

| Metode | Informasi Awal | Akses Awal | Cocok Untuk |
|--------|---------------|-----------|-------------|
| **Black Box** | Hanya URL/IP target | Tidak ada | Simulasi serangan eksternal realistis |
| **Grey Box** | Sebagian: 1 akun user, beberapa dokumen | User biasa | Simulasi insider threat / user yang bocor |
| **White Box** | Penuh: source code, DB schema, semua akun | Admin + DB | Audit keamanan menyeluruh, pre-deployment |

---

## Skenario Tambahan yang Direkomendasikan

Selain 9 skenario utama di atas, berikut skenario khusus berdasarkan **tipe aplikasi**:

| # | Skenario Tambahan | File Tutorial | Mengapa Penting |
|---|------------------|---------------|-----------------|
| 10 | **Aplikasi CMS** (WordPress/Joomla/Drupal) | [10-cms.md](10-cms.md) | ~43% website dunia pakai WordPress; attack surface unik |
| 11 | **SPA + REST/GraphQL API** | [11-spa-api.md](11-spa-api.md) | Arsitektur modern paling umum; JWT & API testing berbeda |
| 12 | **E-Commerce + Payment Gateway** | [12-ecommerce.md](12-ecommerce.md) | Target sensitif; regulasi PCI-DSS; business logic kritis |
| 13 | **Aplikasi Enterprise + SSO/OAuth** | [13-sso-oauth.md](13-sso-oauth.md) | Kerentanan OAuth berdampak ke seluruh ekosistem |
| 14 | **Mobile Backend / API-first App** | [14-mobile-backend.md](14-mobile-backend.md) | APK reverse → API key → server-side attack |

---

## Perbedaan Utama antar Skenario Akses

### Lokal (10.X.X.X)
- **Reconnaissance**: Terbatas pada jaringan lokal; tidak bisa pakai Shodan/OSINT publik
- **Network Discovery**: `nmap` langsung ke subnet; ARP scanning tersedia
- **Kelebihan tester**: Latency rendah, tidak ada firewall cloud/CDN
- **Risiko bisnis**: Lebih rendah tapi bisa sebagai pivot ke infrastruktur lebih dalam
- **Tools tambahan**: `arp-scan`, `netdiscover`, `responder`

### Internal Terbatas (210.57.X.X)
- IP `210.57.x.x` adalah blok IP Indonesia (biasa dipakai universitas/korporat)
- **Reconnaissance**: Bisa sebagian OSINT publik (domain, WHOIS); Shodan mungkin punya data
- **Akses**: Perlu koneksi dari dalam jaringan atau VPN
- **Firewall**: Ada network firewall internal, mungkin IDS/IPS
- **Tools tambahan**: VPN setup, proxy internal

### Internet Publik
- **Reconnaissance**: Full OSINT — Shodan, Censys, crt.sh, Google dorks, Wayback Machine
- **Perhatian hukum**: Paling ketat; wajib ada written authorization
- **CDN/WAF**: Kemungkinan ada Cloudflare, Akamai, atau WAF lain
- **Tools tambahan**: Bypass CDN techniques, WAF evasion

---

## Referensi Pendukung

| File | Konten |
|------|--------|
| [15-durasi-dan-jadwal.md](15-durasi-dan-jadwal.md) | Estimasi waktu + jadwal harian detail untuk semua skenario |
| [16-daftar-tools.md](16-daftar-tools.md) | Daftar lengkap tools + matriks skenario + instalasi |

---

## Rekomendasi Urutan Belajar

```
Pemula:
  Lokal Black Box → Lokal Grey Box → Lokal White Box

Menengah:
  Internal Black Box → Internal Grey Box → Internet Black Box

Lanjutan:
  Internet Grey Box → Internet White Box → Skenario Khusus (10-14)
```

---

## Perbandingan Deliverable per Metode

| Deliverable | Black Box | Grey Box | White Box |
|-------------|-----------|----------|-----------|
| Attack Surface Map | ✓ (terbatas) | ✓ (sedang) | ✓ (lengkap) |
| Vulnerability Report | ✓ | ✓ | ✓ |
| PoC Exploit | ✓ | ✓ | ✓ |
| Source Code Review | ✗ | ✗ | ✓ |
| Architecture Risk | ✗ | Sebagian | ✓ |
| Remediation Detail | Umum | Sedang | Sangat spesifik |
| Waktu rata-rata | 3–5 hari | 5–7 hari | 7–14 hari |
