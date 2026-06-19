# Personnel Pentest: Smishing (SMS Phishing)

**Tujuan:** Test apakah karyawan tertipu SMS yang mengandung link berbahaya  
**Durasi:** 1–3 hari  
**Tools:** SMS gateway, GoPhish (dengan SMS support), atau layanan SMS bulk

---

## Mengapa Smishing Efektif

- Open rate SMS: ~98% (vs email: ~20%)
- Orang lebih percaya SMS daripada email
- SMS pendek → tidak banyak clue untuk verifikasi
- URL di SMS sering dipersingkat (bit.ly) → susah preview
- Nomor bisa di-spoof agar tampak dari operator resmi

---

## Skenario Smishing yang Umum

| Skenario | Pretext | Call to Action |
|----------|---------|----------------|
| Bank OTP | "Kode OTP Anda: [link konfirmasi]" | Klik link, masukkan OTP |
| Paket kiriman | "Paket Anda tertahan, klik untuk konfirmasi" | Klik link |
| Reward/hadiah | "Selamat! Anda memenangkan voucher..." | Klik link, isi data |
| HR Payslip | "Slip gaji November tersedia: [link]" | Klik link, login |
| IT Notifikasi | "Akun Anda akan dikunci: verifikasi sekarang" | Klik link |

---

## Phase 1: Persiapan

```bash
# Setup SMS gateway untuk corporate test
# Opsi legal untuk authorized test:
# - Twilio (sandbox mode)
# - Vonage/Nexmo
# - Lokal: Zenziva, Nusa SMS

# URL shortener untuk tracking klik
# Bisa gunakan GoPhish dengan webhook
# Atau custom: bit.ly dengan analytics, atau setup sendiri

# Buat landing page (clone halaman legitimate)
# Contoh: clone halaman login SSO perusahaan
```

**Prompt Claude Code:**
```
Smishing test untuk targetcorp.co.id (authorized).
Target: semua karyawan dengan nomor HP terdaftar di HR (dari data yang diberikan klien).

Buat:
1. Template SMS (maks 160 karakter) untuk skenario "IT Security Alert"
   - Harus terdengar urgent dan legitimate
   - Sertakan nama perusahaan
   - URL yang dipersingkat ke GoPhish landing page
2. Template SMS untuk skenario "slip gaji"
3. Perbedaan teknik smishing Indonesia vs phishing email
4. Red flags SMS phishing yang harus diajarkan ke karyawan
```

---

## Phase 2: Template SMS

```
Template 1 — IT Security Alert (160 karakter):
"[TargetCorp] Akun SSO Anda terdeteksi login tidak wajar.
Verifikasi segera: http://bit.ly/tc-verify atau akun dikunci 24 jam.
-IT Security"

Template 2 — Slip Gaji:
"[HR TargetCorp] Slip gaji Nov 2024 tersedia.
Download di: http://bit.ly/tc-hr-portal
Login dgn akun karyawan Anda. Info: HRD ext.200"

Template 3 — Paket Internal:
"[Logistik TargetCorp] Ada paket untk Anda di resepsionis.
Konfirmasi penerimaan: http://bit.ly/tc-parcel
Berlaku 24 jam."
```

---

## Phase 3: Tracking & Analisis

```bash
# GoPhish tracking per SMS:
# - Link dibuka (HTTP request ke server)
# - Credential dimasukkan (form submission)
# - Device yang digunakan (User-Agent)
# - Waktu klik

# Yang tidak bisa ditrack di SMS (vs email):
# - SMS tidak bisa pasang tracking pixel
# - Tidak bisa tahu apakah SMS dibaca

# Metrics yang bisa dikumpulkan:
# - Click rate: berapa % yang klik link
# - Submission rate: berapa % yang masukkan credential
# - Time to click: berapa cepat setelah SMS dikirim
# - Device: mobile vs desktop (dari User-Agent)
```

**Prompt Claude Code:**
```
Hasil smishing campaign targetcorp.co.id (200 SMS dikirim):

Klik link: 112/200 (56%)
Credential submitted: 68/200 (34%)
Dilaporkan ke IT: 15/200 (7.5%)

Breakdown per template:
- IT Security Alert: klik 71% (paling tinggi)
- Slip Gaji: klik 55%
- Paket: klik 43%

Breakdown waktu klik setelah SMS dikirim:
- < 5 menit: 45% (sangat impulsif!)
- 5-30 menit: 35%
- > 30 menit: 20%

Buat analisis dan rekomendasi:
1. Mengapa IT Security Alert paling efektif?
2. Tindakan impulsif dalam 5 menit — apa implikasinya untuk training?
3. Perbandingan dengan phishing email (mana lebih berbahaya?)
4. Rekomendasi: MDM policy untuk HP karyawan, SMS filter
5. Training anti-smishing yang efektif dalam 30 menit
```

---

## Phase 4: Edukasi Anti-Smishing

```
Yang diajarkan setelah campaign:

CIRI SMS PHISHING:
✗ Menciptakan urgensi: "segera", "24 jam", "akan dikunci"
✗ Link yang dipersingkat (bit.ly, tinyurl)
✗ Minta klik link untuk login
✗ Mengaku dari bank/instansi/IT tapi nomor tidak dikenal
✗ Hadiah tidak terduga

CARA AMAN:
✓ Jangan klik link dari SMS — buka aplikasi/website langsung
✓ Verifikasi via saluran resmi (nomor resmi bank/IT helpdesk)
✓ Preview URL sebelum klik (tekan lama di HP)
✓ Laporkan SMS mencurigakan ke IT Security
✓ Aktifkan filter spam SMS di HP
```

---

## Checklist Smishing Pentest

```
[ ] Dapatkan daftar nomor HP target dari klien (bukan scraping)
[ ] Setup SMS gateway (harus legal dan traceable)
[ ] Buat landing page + tracking
[ ] Buat 2-3 template SMS yang berbeda
[ ] Kirim bertahap (tidak sekaligus)
[ ] Pantau klik dan submission real-time
[ ] Reveal dalam 48-72 jam setelah pengiriman
[ ] Edukasi singkat via SMS follow-up
[ ] Laporan: click rate, submission rate, waktu respon
[ ] Rekomendasi: MDM, SMS filter, training
```

---

## Durasi Estimasi

| Fase | Durasi |
|------|-----------|----------|
| Persiapan | 0.5–1 hari | 1–2 hari |
| Eksekusi + monitoring | 2–3 hari | 2–3 hari (sama) |
| Laporan + debrief | 0.5 hari | 1–2 hari |
| **Total** | **3–5 hari** | **4–7 hari** |
