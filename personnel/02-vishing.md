# Personnel Pentest: Vishing (Voice Phishing)

**Tujuan:** Test apakah karyawan bisa dimanipulasi via telepon untuk memberikan informasi sensitif  
**Durasi:** 1–3 hari  
**Peralatan:** Telepon/VOIP, voice changer (opsional), script

---

## Skenario Vishing yang Umum

| Skenario | Penyerang berpura jadi | Target informasi |
|----------|----------------------|-----------------|
| IT Support | Teknisi IT yang perlu "verifikasi" | Username + password |
| Vendor / Supplier | Petugas dari vendor terkenal | Data rekening bank, PO |
| Bank / OJK | Petugas bank atau OJK | Data nasabah, PIN |
| CEO / Atasan | Direktur / manajemen | Transfer dana, data |
| Survey Resmi | Petugas survey pemerintah | Data pribadi karyawan |

---

## Phase 1: Reconnaissance Sebelum Menelepon

```
Yang perlu diketahui sebelum menelepon:
- Nama karyawan target (dari LinkedIn, website, email list)
- Nama atasan target (dari organogram)
- Nama vendor/supplier yang biasa berinteraksi
- Istilah teknis yang relevan untuk posisi target
- Sistem yang digunakan (dari job posting, forum, OSINT)
- Nomor tiket/referensi yang terdengar valid
- Jam sibuk vs jam santai di kantor target
```

**Prompt Claude Code:**
```
Persiapan vishing test untuk targetcorp.co.id (authorized).
Target: staf Finance departemen.

Dari OSINT:
- Software yang digunakan: SAP Finance, Microsoft 365
- IT Helpdesk number: ext. 100
- IT Manager: Budi Santoso
- Finance Manager: Ibu Dewi
- Jam sibuk: 09:00-11:00 dan 14:00-16:00

Buatkan:
1. Script vishing untuk skenario "IT Support reset password"
2. Pertanyaan verifikasi yang terdengar legitimate
3. Jawaban untuk kemungkinan pertanyaan balik dari target
4. Tanda-tanda target mulai curiga dan cara mengatasinya
5. Cara mengakhiri percakapan dengan natural
```

---

## Phase 2: Script Vishing — IT Support

```
SCRIPT: IT Support Reset Password

[TESTER menelepon target]

T: "Selamat pagi, saya hubungi dari IT Helpdesk. Bisa berbicara dengan
    [nama karyawan]?"

K: "Ya, saya sendiri."

T: "Selamat pagi Pak/Bu [nama]. Saya [nama palsu] dari IT Support.
    Nomor tiket saya IT-2024-1847.
    Kami baru saja mendeteksi aktivitas login yang tidak biasa pada
    akun Anda. Beberapa login gagal dari lokasi yang tidak dikenal.
    Untuk keamanan akun Anda, saya perlu membantu reset password
    segera sebelum akun Anda dikunci secara otomatis."

K: "Oh, ya? Dari mana lokasinya?"

T: "Sistem kami mendeteksi dari IP Surabaya, sekitar pukul 23:00 tadi
    malam. Akun email dan SAP Anda mencoba diakses. Untuk prosedur
    keamanan, saya perlu verifikasi identitas Anda dulu.
    Bisa sebutkan username Anda?"

K: [menyebutkan username]

T: "Terima kasih. Untuk verifikasi terakhir, saya perlu konfirmasi
    password lama Anda sebelum kami reset. Ini prosedur standar kami."

[JIKA TARGET MEMBERIKAN PASSWORD → BERHASIL, catat]

[JIKA TARGET MENOLAK:]
T: "Pak/Bu, ini memang terdengar tidak biasa, tapi ini prosedur
    resmi kami saat ada potensi compromise. Jika tidak diverifikasi
    dalam 10 menit, sistem akan otomatis lock akun Anda dan butuh
    2 hari kerja untuk dibuka kembali."

[SETELAH TEST SELESAI, segera reveal dan edukasi]
```

---

## Phase 3: Skenario CEO Fraud (Business Email Compromise)

```
SCRIPT: Direktur meminta transfer mendesak

T: "Halo, ini [nama asli CFO/CEO] ya?"

K: "Ya betul."

T: "Saya [nama CEO palsu]. Maaf ganggu, saya sedang di meeting
    penting dengan calon investor dari Singapura.
    Saya butuh transfer dana mendesak ke rekening mereka hari ini
    sebelum pukul 15:00. Totalnya 250 juta.
    Tolong proses sekarang, saya kirimkan detailnya via WA ya.
    Jangan beritahu siapa-siapa dulu karena ini masih confidential."

[Ini adalah skenario klasik BEC — catat apakah target langsung ikut
atau memverifikasi terlebih dahulu]
```

---

## Phase 4: Dokumentasi Hasil

```
Yang harus dicatat setiap panggilan:

- Nama target & departemen
- Waktu panggilan
- Skenario yang digunakan
- Informasi apa yang berhasil didapat
- Berapa lama sebelum target curiga
- Apakah target verifikasi identitas penelepon?
- Apakah target menolak dan kenapa?
- Apakah target melaporkan ke IT/security?
```

**Prompt Claude Code:**
```
Hasil vishing campaign targetcorp.co.id:

20 panggilan dilakukan:
- Berhasil dapatkan username: 12/20 (60%)
- Berhasil dapatkan password: 6/20 (30%)
- Target meminta verifikasi identitas penelepon: 4/20 (20%)
- Target menolak tegas sejak awal: 3/20 (15%)
- Target melaporkan ke IT Security: 2/20 (10%)

Paling rentan: staf Finance dan HR
Paling waspada: staf IT

Buatkan:
1. Analisis mendalam kenapa Finance paling rentan
2. Rekomendasi training anti-vishing spesifik
3. Prosedur verifikasi yang seharusnya dilakukan karyawan
4. Kebijakan yang perlu dibuat: transfer request via telepon harus verifikasi via jalur lain
5. Template laporan untuk CISO
```

---

## Panduan Anti-Vishing untuk Training Karyawan

```
TIDAK PERNAH berikan via telepon:
✗ Password (IT asli TIDAK PERNAH minta password)
✗ PIN ATM / OTP
✗ Data kartu kredit perusahaan
✗ Transfer dana tanpa verifikasi ganda

SELALU lakukan:
✓ Minta nomor tiket/referensi dan nama penelepon
✓ Tutup telepon → hubungi IT Helpdesk via nomor resmi untuk verifikasi
✓ Untuk request transfer: konfirmasi via email + telepon ke nomor yang sudah dikenal
✓ Laporkan panggilan mencurigakan ke IT Security
```

---

## Checklist Vishing Pentest

```
[ ] Scope jelas: siapa yang boleh ditarget, skenario apa
[ ] OSINT target: nama, posisi, sistem yang digunakan
[ ] Buat script untuk setiap skenario
[ ] Siapkan jawaban untuk pertanyaan balik
[ ] Eksekusi panggilan, dokumentasikan setiap call
[ ] Catat: informasi yang berhasil didapat per panggilan
[ ] Reveal dan edukasi setelah selesai
[ ] Laporan: tingkat keberhasilan + rekomendasi training
[ ] Usulkan prosedur verifikasi telepon yang baru
```

---

## Durasi Estimasi

| Fase | Durasi |
|------|-----------|----------|
| Persiapan script + OSINT | 0.5–1 hari | 1–2 hari |
| Eksekusi (20 panggilan) | 1–2 hari | 1–2 hari (sama) |
| Analisis + laporan | 0.5–1 hari | 1–2 hari |
| **Total** | **2–4 hari** | **3–6 hari** |
