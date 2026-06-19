# Personnel Pentest: Pretexting & Impersonation

**Tujuan:** Test apakah karyawan bisa dimanipulasi oleh orang yang berpura-pura jadi pihak lain  
**Durasi:** 2–4 hari  
**Metode:** Kombinasi fisik, telepon, dan email dengan narasi yang disiapkan

---

## Perbedaan dengan Skenario Lain

| Aspek | Vishing | Pretexting |
|-------|---------|------------|
| Media | Telepon saja | Multi-channel (telepon + email + fisik) |
| Kompleksitas | Satu call | Narasi berlapis, berhari-hari |
| Target informasi | Credential | Akses sistem, data bisnis, tindakan spesifik |
| Durasi | Menit | Hari hingga minggu |

---

## Skenario Pretexting Umum

| Skenario | Penyerang sebagai | Tujuan |
|----------|------------------|--------|
| Auditor eksternal | Konsultan/auditor dari KAP ternama | Akses ke dokumen finansial |
| Regulator | Petugas OJK/BI/Kemkominfo | Data compliance, log sistem |
| Vendor baru | Representatif vendor software/hardware | Akses demo ke sistem produksi |
| Jurnalis | Reporter dari media bisnis | Informasi internal perusahaan |
| Headhunter | Recruiter dari perusahaan besar | Data struktur organisasi dan salary |

---

## Phase 1: Persiapan Pretext

```
Elemen pretext yang meyakinkan:

1. Identitas yang konsisten:
   - Nama: terdengar natural untuk posisi tersebut
   - Perusahaan: mirip dengan perusahaan nyata (bukan clone persis)
   - Email: domain yang terlihat legitimate
   - Nomor telepon: dapat dihubungi balik (setup VoIP)
   - LinkedIn: profil sederhana tapi existing

2. "Props" pendukung:
   - Email konfirmasi yang terlihat resmi
   - "Surat penugasan" dengan letterhead
   - Business card
   - Website sederhana untuk konsultan fiktif

3. Riset mendalam:
   - Istilah industri yang tepat
   - Nama pemimpin perusahaan yang nyata
   - Isu yang sedang relevan di industri target
   - Nama-nama vendor/klien yang sering disebut
```

**Prompt Claude Code:**
```
Pretexting test untuk targetcorp.co.id (perusahaan manufaktur, authorized).
Pretext: auditor dari KAP fiktif "Wijaya & Rekan" yang sedang audit tahunan.

Dari OSINT:
- Direktur Keuangan: Bapak Hendra Gunawan
- Auditor KAP asli mereka: Ernst & Young (dari annual report 2022)
- Sistem akuntansi: SAP
- Tahun fiskal: Januari-Desember

Buatkan:
1. Narasi pretexting yang lengkap (timeline 3 hari)
2. Email awal untuk "memperkenalkan diri" ke staf Finance
3. Daftar dokumen/akses yang akan diminta (realistis untuk auditor)
4. Pertanyaan teknis yang terdengar legitimate untuk auditor akuntansi
5. Red flags yang seharusnya membuat karyawan curiga
```

---

## Phase 2: Multi-Stage Pretexting (Contoh 3 Hari)

```
HARI 1 — Establish Contact

Email pertama (dari alamat wijayarekan@domain-mirip.com):

"Kepada Yth. Tim Finance PT Target Corp,

Sehubungan dengan penugasan audit tahunan yang telah disetujui
Direksi, kami dari KAP Wijaya & Rekan akan memulai fieldwork
pada pekan ini.

Auditor kami yang ditugaskan:
- Senior Auditor: [nama tester]
- Junior Auditor: [nama tester 2]

Mohon dipersiapkan dokumen berikut:
1. Trial balance per November 2024
2. Daftar aset tetap
3. Rekonsiliasi bank 3 bulan terakhir

Kami akan menghubungi Bapak Hendra melalui telepon besok pagi.

Hormat kami,
KAP Wijaya & Rekan"

---
HARI 2 — Establish Trust via Phone

Telepon ke staf Finance:
"Selamat pagi, saya [nama] dari KAP Wijaya. 
Kemarin sudah kirim email ke tim Anda.
Apakah Bapak Hendra sudah informasikan mengenai audit kami?
[Jika ya] Baik, kami butuh akses ke modul SAP untuk verifikasi..."

---
HARI 3 — Escalate Request

"Sebagai tindak lanjut, kami perlu akses read-only ke SAP FI/CO
untuk verifikasi jurnal. Biasanya prosedurnya bagaimana di sini?"
```

---

## Phase 3: Impersonasi Regulator

```
Skenario: Petugas OJK (Otoritas Jasa Keuangan)

Script telepon:
"Selamat pagi, ini [nama] dari OJK Kantor Regional III.
Saya ingin berbicara dengan Compliance Officer Anda.
Kami sedang melakukan pemeriksaan rutin terkait implementasi
POJK No. XX tentang [topik relevan].

Kami membutuhkan data berikut dalam 2 jam:
- Log transaksi 30 hari terakhir
- Daftar pengguna sistem dengan akses level
- Laporan insiden keamanan bulan ini

Ini mendesak — ada deadline dari kantor pusat OJK Jakarta."

CATATAN ANALISIS:
- Apakah target meminta nomor surat resmi?
- Apakah target memverifikasi ke OJK langsung?
- Apakah target konsultasi ke atasan?
- Berapa cepat target bereaksi terhadap "urgensi"?
```

---

## Phase 4: Analisis Kerentanan

**Prompt Claude Code:**
```
Pretexting test (3 skenario, masing-masing 3 percobaan):

Skenario Auditor KAP:
- 2/3 berhasil: staf Finance memberikan akses ke shared folder dokumen
- 1/3: staf meminta konfirmasi ke CFO sebelum share dokumen

Skenario Vendor Demo:
- 3/3 berhasil: IT memberikan demo account ke "vendor"
- Tidak ada yang verifikasi ke procurement/manajemen

Skenario Regulator OJK:
- 1/3 berhasil: compliance officer langsung menyiapkan data
- 2/3: meminta surat resmi terlebih dahulu

Analisis:
1. Mengapa vendor scenario paling berhasil (100%)?
2. Pola: urgensi + otoritas = kepatuhan (Cialdini's principles)?
3. Departemen IT paling rentan untuk skenario apa?
4. Rekomendasi prosedur verifikasi multi-layer
5. Training scenario-based yang efektif
```

---

## Phase 5: Social Engineering Principles (Untuk Training)

```
6 Prinsip Persuasi Cialdini yang Dieksploitasi Penyerang:

1. OTORITAS — berpura-pura jadi figur otoritas (auditor, regulator, CEO)
   → Verifikasi: selalu konfirmasi identitas via saluran independen

2. URGENSI/SCARCITY — "harus selesai dalam 2 jam"
   → Red flag: permintaan mendadak selalu harus diverifikasi

3. LIKING — membangun rapport, bersikap ramah
   → Jangan biarkan keramahan menurunkan kewaspadaan

4. SOCIAL PROOF — "tim lain sudah memberikan aksesnya"
   → Verifikasi ke kolega yang disebutkan

5. RECIPROCITY — memberikan sesuatu dulu agar kita merasa berhutang
   → Hadiah kecil sebelum request besar adalah tanda bahaya

6. COMMITMENT — membuat kita setuju hal kecil dulu
   → "Tadi Anda bilang bisa bantu, sekarang saya butuh..."
```

---

## Checklist Pretexting Pentest

```
[ ] Setup identitas yang konsisten (email, telepon, LinkedIn minimal)
[ ] Riset mendalam tentang target dan konteks bisnis
[ ] Buat narasi multi-stage yang masuk akal
[ ] Dokumentasikan setiap interaksi (rekaman jika diizinkan)
[ ] Test minimal 3 skenario berbeda
[ ] Catat: siapa yang curiga, siapa yang percaya, siapa yang verifikasi
[ ] Reveal dan debrief setelah semua test selesai
[ ] Laporan: analisis psikologis + rekomendasi prosedur verifikasi
[ ] Training: simulasi pretexting untuk semua departemen
```

---

## Durasi Estimasi

| Fase | Durasi |
|------|-----------|----------|
| Persiapan pretext + OSINT | 1–2 hari | 3–5 hari |
| Eksekusi (multi-hari) | 3–5 hari | 3–5 hari (sama) |
| Laporan + analisis psikologis | 1 hari | 2–3 hari |
| **Total** | **5–8 hari** | **8–13 hari** |
