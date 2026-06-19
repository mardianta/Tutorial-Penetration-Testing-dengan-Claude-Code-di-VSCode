# Personnel Pentest: Phishing Email Campaign

**Tujuan:** Mengukur seberapa banyak karyawan yang tertipu email phishing  
**Durasi:** 1–2 minggu (termasuk persiapan, eksekusi, dan analisis)  
**Tools:** GoPhish, SET (Social Engineering Toolkit), King Phisher

---

## Skenario Phishing yang Umum

| Skenario | Pretext | Target Aksi |
|----------|---------|------------|
| IT Reset Password | Email dari "IT Support" → link login palsu | Memasukkan credential |
| HR Dokumen | Email dari "HRD" → "lihat slip gaji Anda" | Download malware |
| Finance Invoice | Email dari "vendor" → invoice dengan macro | Buka file Office berbahaya |
| CEO Fraud | Email dari "Direktur" → minta transfer dana | Tindakan finansial |
| Parcel Tracking | Email dari "JNE/J&T" → klik untuk tracking | Klik link berbahaya |

---

## Phase 1: Persiapan & OSINT Target

```bash
# Kumpulkan info target dari OSINT
# Email format perusahaan
# Linkedin: nama karyawan + departemen
# Website: struktur organisasi, nama pimpinan

# theHarvester — kumpulkan email karyawan
theHarvester -d targetcorp.co.id -b google,linkedin,bing

# hunter.io — verifikasi format email (jika dalam scope)
# Format umum: firstname.lastname@company.com
#              f.lastname@company.com
#              firstname@company.com

# LinkedIn scraping (manual atau via tools)
# Identifikasi: nama CEO, nama IT Head, nama HR Manager

# Buat email list target sesuai scope
cat > targets.csv << 'EOF'
Email,First Name,Last Name,Department
john.doe@targetcorp.co.id,John,Doe,Finance
jane.smith@targetcorp.co.id,Jane,Smith,HR
IT_staff@targetcorp.co.id,IT,Staff,IT
EOF
```

**Prompt Claude Code:**
```
Saya akan menjalankan phishing campaign untuk targetcorp.co.id (authorized).
Departemen target: Finance, HR, IT.

Dari OSINT:
- Email format: firstname.lastname@targetcorp.co.id
- Direktur Keuangan: Budi Santoso
- IT Manager: Dewi Rahayu
- Saat ini: menjelang akhir bulan (waktu invoice/gaji)

Buatkan:
1. 3 skenario phishing yang paling relevan untuk konteks perusahaan Indonesia
2. Template email untuk skenario paling efektif (bahasa Indonesia)
3. Pretext yang masuk akal untuk departemen Finance
4. Red flags yang seharusnya membuat karyawan curiga (untuk training)
```

---

## Phase 2: Setup GoPhish

```bash
# Install GoPhish
wget https://github.com/gophish/gophish/releases/download/v0.12.1/gophish-v0.12.1-linux-64bit.zip
unzip gophish-*.zip && chmod +x gophish

# Jalankan GoPhish
./gophish &
# Admin UI: https://localhost:3333
# Login default: admin/gophish

# Konfigurasi:
# 1. Sending Profile: SMTP server untuk kirim email
# 2. Landing Page: halaman phishing (clone dari halaman login asli)
# 3. Email Template: template email phishing
# 4. Users & Groups: daftar target
# 5. Campaign: gabungkan semua + jadwalkan
```

### Clone Landing Page Login

```bash
# GoPhish bisa clone halaman secara otomatis
# Atau manual:
wget -P phishing_site/ https://mail.targetcorp.co.id/owa/auth/logon.aspx
# Edit file → ganti action form ke server GoPhish
# Tambahkan: kredensial yang dimasukkan akan dikirim ke server kita

# Atau gunakan SET
setoolkit
1 → Social-Engineering Attacks
2 → Website Attack Vectors
3 → Credential Harvester Attack Method
2 → Site Cloner
Enter URL to clone: https://mail.targetcorp.co.id
```

---

## Phase 3: Email Template (Contoh Indonesia)

```
Template 1: IT Password Reset

From: it-support@targetcorp-helpdesk.co.id  ← domain mirip tapi beda
Subject: [URGENT] Password Anda akan expired dalam 24 jam

Kepada Yth. [First Name],

Sistem keamanan kami mendeteksi bahwa password akun Anda
(john.doe@targetcorp.co.id) akan kadaluarsa dalam 24 jam.

Untuk menjaga keamanan akun Anda, segera lakukan pembaruan password
melalui link berikut:

🔗 [Perbarui Password Sekarang](http://phishing.attacker.com/reset)

Jika Anda tidak melakukan pembaruan dalam 24 jam, akses email dan
sistem internal Anda akan dinonaktifkan sementara.

Hormat kami,
Tim IT Support
PT Target Corp Indonesia

---
Pesan ini dikirim otomatis oleh sistem. Jangan membalas email ini.
IT Helpdesk: ext. 100 | it-support@targetcorp.co.id
```

```
Template 2: HR Slip Gaji

From: hrd-noreply@targetcorp.co.id  ← email spoofed (jika SPF lemah)
Subject: Slip Gaji Bulan November 2024 - Mohon Didownload

[First Name] yang terhormat,

Slip gaji Anda untuk bulan November 2024 telah tersedia.
Silakan download melalui portal HR berikut:

[Klik di sini untuk download Slip Gaji]
http://phishing.attacker.com/hr-portal

Mohon konfirmasi penerimaan dengan login menggunakan
akun karyawan Anda.

Terima kasih,
Departemen HRD
PT Target Corp Indonesia
```

---

## Phase 4: Eksekusi Campaign

```bash
# Di GoPhish dashboard:
# 1. Buat campaign baru
# 2. Pilih email template, landing page, target group, sending profile
# 3. Jadwalkan: kirim bertahap (tidak sekaligus → tidak trigger spam filter)
#    - Batch 1: pukul 09:00 (saat karyawan baru masuk)
#    - Batch 2: pukul 13:30 (setelah makan siang)

# Pantau metrics real-time:
# - Emails Sent
# - Emails Opened (tracking pixel)
# - Links Clicked
# - Credentials Submitted
# - Reports (karyawan yang melaporkan email mencurigakan)
```

**Prompt Claude Code:**
```
GoPhish campaign setelah 48 jam:

Hasil:
- Emails sent: 150
- Opened: 98 (65.3%)
- Clicked link: 67 (44.7%)
- Submitted credentials: 34 (22.7%)
- Reported to IT: 8 (5.3%)

Breakdown per departemen:
- Finance: 15/30 submitted (50%) ← paling rentan
- IT: 3/20 submitted (15%)
- HR: 8/25 submitted (32%)
- Operations: 8/75 submitted (10.7%)

Analisis hasil dan buat:
1. Benchmark: apakah angka ini di atas/bawah rata-rata industri?
2. Departemen mana yang butuh training paling urgen?
3. Rekomendasi training yang spesifik per departemen
4. Metrik sukses untuk program security awareness
5. Format laporan untuk manajemen (non-teknis)
```

---

## Phase 5: Debrief & Training

```
Setelah campaign selesai:

1. Kirim email "reveal" ke semua target:
   "Email yang Anda terima pada [tanggal] adalah bagian dari
   security awareness test yang dilakukan oleh tim IT Security.
   Berikut tanda-tanda phishing yang seharusnya Anda perhatikan:..."

2. Sesi training untuk departemen yang paling rentan

3. Tips mengenali phishing:
   - Cek domain pengirim (bukan hanya display name)
   - Hover link sebelum klik
   - Jangan masukkan credential jika ragu
   - Laporkan ke IT jika curiga
   - Verifikasi via telepon jika request unusual
```

---

## Checklist Phishing Campaign

```
[ ] Dapatkan scope yang jelas (departemen mana, jenis pretext apa)
[ ] OSINT: kumpulkan email dan nama karyawan
[ ] Setup GoPhish + landing page (clone login page target)
[ ] Buat template email yang relevan dan realistis
[ ] Setup tracking (pixel open + link click + form submit)
[ ] Kirim bertahap (hindari spam filter)
[ ] Monitor real-time, catat semua data
[ ] Setelah selesai: kirim reveal email
[ ] Debrief dan training untuk yang "jatuh"
[ ] Laporan: metrics + rekomendasi + training plan
```

---

## Durasi Estimasi

| Fase | Durasi |
|------|-----------|----------|
| Persiapan (OSINT + setup + template) | 1–2 hari | 3–4 hari |
| Eksekusi campaign | 2–5 hari | 2–5 hari (sama) |
| Analisis + laporan | 1 hari | 2–3 hari |
| **Total** | **4–8 hari** | **7–12 hari** |
