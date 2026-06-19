# Personnel Pentest: Physical Intrusion & Tailgating

**Tujuan:** Test apakah penyerang bisa memasuki area terbatas secara fisik  
**Durasi:** 1–3 hari  
**Peralatan:** Kamera tersembunyi (legal), alat tulis, pakaian yang tepat, ID palsu (dalam scope)

---

## ⚠️ Persiapan Wajib

```
SEBELUM MEMULAI:
✓ Surat otorisasi + get-out-of-jail letter yang ditandatangani
✓ Nomor kontak darurat klien (jika tester ditangkap satpam)
✓ Konfirmasi dengan security manager (tapi BUKAN dengan satpam lapangan)
✓ Foto ID tester yang sah (untuk menunjukkan ke satpam jika diperlukan)
✓ Tentukan batas: area mana yang boleh dimasuki, apa yang boleh dilakukan
```

---

## Skenario Physical Intrusion

| Skenario | Teknik | Target |
|----------|--------|--------|
| Tailgating | Ikut di belakang karyawan yang swipe kartu | Masuk gedung |
| Impersonasi teknisi | Berpura-pura jadi teknisi AC/IT/maintenance | Masuk server room |
| Impersonasi delivery | Membawa paket/kotak | Masuk ke dalam |
| Visitor abuse | Daftar sebagai tamu, lalu eksplorasi sendiri | Area terbatas |
| Dumpster diving | Geledah tempat sampah | Dokumen sensitif |

---

## Phase 1: Reconnaissance Fisik

```
Lakukan sebelum mencoba masuk:

1. Observasi dari luar gedung
   - Berapa banyak pintu masuk?
   - Jenis access control: swipe card, PIN, wajah, atau satpam manual?
   - Jam sibuk (banyak orang masuk → tailgating lebih mudah)?
   - Apakah ada kamera CCTV di pintu masuk?

2. Pelajari seragam/penampilan karyawan
   - Warna seragam (jika ada)
   - Apakah pakai ID card terlihat?
   - Formal atau kasual?

3. Vendor/tamu yang sering datang
   - Jam berapa biasanya ada delivery?
   - Vendor IT mana yang sering datang?
   - Apakah ada loket tamu yang perlu registrasi?
```

---

## Phase 2: Tailgating Test

```
Teknik tailgating:

BASIC:
- Tunggu di dekat pintu saat banyak karyawan masuk
- Saat ada yang buka pintu, ikut masuk sambil pura-pura pegang HP
- Hindari eye contact
- Berjalan dengan percaya diri seolah sudah biasa

INTERMEDIATE:
- Bawa "dokumen" atau map yang terlihat official
- Pura-pura sedang telepon → tidak perlu menjawab pertanyaan
- Jika ditanya: "Saya ada meeting dengan [nama yang terdengar familiar]"

ADVANCED:
- Koordinasi dengan 2 tester: tester 1 distraksi satpam, tester 2 masuk
- Timing saat shift pergantian satpam (konsentrasi berkurang)
- Saat ada karyawan banyak masuk bersamaan (morning rush)

DOKUMENTASI:
- Catat: apakah ada yang menegur? Berapa kali tailgating berhasil?
- Jangan rekam karyawan tanpa izin — rekam hanya proses akses kontrol
```

**Prompt Claude Code:**
```
Physical intrusion test di kantor pusat targetcorp.co.id (authorized).

Observasi:
- Gedung 12 lantai, 4 pintu masuk
- Access control: swipe card di pintu utama
- Satpam: 2 orang di lobby, shift 8 jam
- Jam 08:30-09:00: sangat ramai, banyak karyawan masuk bersamaan
- ID card: tidak semua karyawan memakai ID card terlihat

Hasil test hari 1:
- Tailgating berhasil 4/5 percobaan
- 1 percobaan ditegur satpam: "Maaf Bapak, bisa tunjukkan ID?"
- Berhasil masuk hingga lantai 3 (area operasional)

Analisis:
1. Kelemahan utama yang ditemukan
2. Kenapa 80% tailgating berhasil?
3. Apa yang bisa dilakukan penyerang setelah masuk? (skenario terburuk)
4. Rekomendasi: mantrapping, anti-tailgating door, security awareness
5. Standar ISO 27001 yang relevan untuk physical security
```

---

## Phase 3: Impersonasi Teknisi

```
Persiapan impersonasi:

1. Pilih vendor yang umum: Schneider Electric (UPS), Daikin (AC),
   Cisco (network), atau jasa cleaning service

2. Siapkan:
   - Seragam atau pakaian yang sesuai
   - Toolbox atau peralatan (kosong atau isi minimal)
   - "Work order" yang terlihat resmi (kertas berlogo vendor)
   - Nama yang familiar dari vendor tersebut

3. Script untuk resepsionis:
   "Selamat pagi, saya dari Schneider Electric.
    Kami ada jadwal maintenance UPS di server room lantai B2.
    Ini work order kami. Bisa saya diantar ke sana?"

4. Dokumentasikan:
   - Apakah satpam/resepsionis verifikasi ke vendor asli?
   - Apakah diminta tunjukkan ID vendor?
   - Berapa lama sebelum diperbolehkan masuk?
   - Apakah ada escort? (seharusnya ada)
```

---

## Phase 4: Dumpster Diving

```bash
# Cari dokumen sensitif yang dibuang
# Lakukan di area tempat sampah yang bisa diakses tester

Yang sering ditemukan:
- Printout email dengan informasi sensitif
- Slip gaji
- Diagram jaringan/topologi
- Username/password yang ditulis di kertas
- Dokumen NDA atau kontrak
- Badge/ID card yang dibuang tapi tidak dihancurkan

Dokumentasikan semua temuan (foto saja, jangan bawa)
Catat: jenis dokumen, tingkat sensitivitas
```

---

## Phase 5: Eksplorasi Setelah Masuk

```
Setelah berhasil masuk, test sampai mana bisa masuk:

[ ] Lobby → lantai umum → lantai terbatas
[ ] Apakah ada pintu server room yang terbuka/tidak terkunci?
[ ] Apakah bisa akses komputer yang ditinggal tidak ter-lock?
[ ] Apakah ada whiteboard berisi informasi sensitif?
[ ] Apakah ada post-it note berisi password di monitor?
[ ] Apakah bisa colokkan USB ke komputer yang ditinggal?
[ ] Apakah ada dokumen sensitif yang terbuka di meja?

CATATAN: Jangan sentuh atau ambil apapun. Dokumentasikan hanya dengan foto.
```

---

## Checklist Physical Intrusion

```
[ ] Get-out-of-jail letter ready
[ ] Reconnaissance fisik dari luar (minimal 1 hari)
[ ] Test tailgating di berbagai waktu (pagi, siang)
[ ] Test impersonasi vendor/teknisi
[ ] Dokumentasikan setiap percobaan: berhasil/gagal, alasan
[ ] Dumpster diving di area yang diizinkan
[ ] Catat: area mana yang berhasil dimasuki
[ ] Foto bukti (tanpa wajah karyawan)
[ ] Reveal kepada security manager setelah selesai
[ ] Laporan: physical security gaps + rekomendasi
```

---

## Rekomendasi untuk Laporan

**Prompt Claude Code:**
```
Hasil physical intrusion test:
- Tailgating: berhasil 16/20 percobaan (80%)
- Impersonasi teknisi: berhasil masuk server room (tidak ada verifikasi ke vendor)
- Komputer tidak ter-lock: ditemukan 8 komputer ditinggal tanpa screen lock
- Post-it password: 3 ditemukan di meja karyawan
- Dumpster: ditemukan printout email konfidensial dan diagram network

Buatkan laporan fisik security yang mencakup:
1. Executive summary (untuk manajemen senior)
2. Foto/bukti yang bisa ditampilkan (anonim)
3. Rekomendasi teknis per temuan
4. Security awareness training yang diperlukan
5. KPI untuk mengukur improvement (target: tailgating < 5% dalam 6 bulan)
6. Referensi ISO 27001:2022 untuk physical security controls
```

---

## Durasi Estimasi

| Fase | Durasi |
|------|-----------|----------|
| Reconnaissance | 1 hari | 1–2 hari |
| Test intrusion (2–3 hari) | 2–3 hari | 2–3 hari (sama) |
| Laporan | 0.5–1 hari | 2–3 hari |
| **Total** | **3–5 hari** | **5–8 hari** |
