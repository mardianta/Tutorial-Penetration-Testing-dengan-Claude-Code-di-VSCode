# Hardware Pentest: ATM & POS Terminals

**Target:** ATM (Automated Teller Machine), POS (Point of Sale) terminal  
**Durasi:** 3–5 hari  
**Regulasi:** PCI-DSS wajib dipenuhi; koordinasi ketat dengan bank/payment processor

---

## ⚠️ Disclaimer Penting

```
ATM dan POS pentest memerlukan:
1. Izin tertulis dari bank/payment processor pemilik ATM
2. Koordinasi dengan tim keamanan bank
3. Dilakukan pada mesin uji (test unit), BUKAN mesin produksi
4. Comply dengan PCI-DSS PA-DSS dan PCI-P2PE
5. Pengawasan engineer bank selama pengujian

Melakukan testing pada ATM tanpa izin = tindak pidana berat
```

---

## Arsitektur ATM

```
[Nasabah]
    ↓ (card + PIN)
[ATM Hardware]
  ├── Card Reader (magnetic stripe + EMV chip)
  ├── PIN Pad (HSM — Hardware Security Module)
  ├── Cash Dispenser
  ├── Receipt Printer
  └── Kamera
    ↓ (network)
[ATM Controller PC] — Windows XP/7/10 embedded
    ↓ (TCP/IP)
[Bank Backend / ATM Switch]
    ↓
[Core Banking System]
```

---

## Phase 1: Physical Security Assessment

```bash
# Inspeksi fisik ATM (tanpa membuka mesin)

Checklist fisik:
[ ] Cek card slot — ada skimmer overlay?
[ ] Cek PIN pad — ada overlay atau kamera tersembunyi?
[ ] Cek kamera ATM — apakah bisa lihat nasabah masukkan PIN?
[ ] Cek housing ATM — ada bekas dibuka paksa?
[ ] USB port — apakah tertutup / dilindungi?
[ ] Apakah ATM terkunci ke tembok/lantai dengan benar?
[ ] Segel keamanan — masih utuh?
```

---

## Phase 2: Network Connectivity Testing

```bash
# ATM terhubung ke bank via jaringan — bisa private network atau internet

# Scan port dari posisi tester (jika dapat akses ke jaringan ATM)
nmap -sV 192.168.ATM.IP

# Port yang umum digunakan ATM:
# 9100 — XFS/CEN protokol (internal)
# 4000, 8583 — ISO 8583 (ATM transaction)
# 623 — IPMI (management)
# 3389 — RDP (remote management) ← sering tidak terlindungi!
# 80/443 — update server atau management web

# Test apakah RDP exposed
nmap -p 3389 192.168.ATM.0/24
```

**Prompt Claude Code:**
```
ATM pentest dalam lingkungan lab bank (authorized).
Scan ATM controller PC:

OS: Windows 7 Embedded Standard (EOL!)
Port terbuka:
- 3389/tcp RDP → berhasil connect tanpa auth! ← KRITIS
- 445/tcp SMB → EternalBlue vulnerable (MS17-010)
- 80/tcp IIS 7.5 → management interface

RDP login dengan credential default "ATMAdmin:ATMAdmin" berhasil.
Desktop Windows terlihat — ada shortcut XFS ATM software.

Analisis:
1. Windows 7 EOL di ATM — berapa banyak CVE yang applicable?
2. EternalBlue pada ATM — dampak jika dikuasai?
3. RDP tanpa proteksi → bisa akses ATM controller dari jaringan bank?
4. Dari akses Windows, apakah bisa trigger cash dispense?
5. Rekomendasi hardening ATM network
```

---

## Phase 3: ATM Black Box Attack Simulation

```bash
# Black box attack — tanpa akses ke dalam ATM
# Simulasikan penyerang yang punya akses ke jaringan ATM

# 1. Network MITM pada koneksi ATM ke bank
# (perlu posisi di jaringan antara ATM dan bank)
arpspoof -i eth0 -t ATM_IP Bank_Gateway_IP

# Capture dan analisis transaksi (untuk test, bukan produksi)
wireshark -i eth0 -k -w atm_traffic.pcap

# 2. Replay attack — apakah transaksi bisa di-replay?
# (harus dilakukan di environment test dengan uang test)

# 3. Transaction manipulation — apakah bisa ubah jumlah?
# (hanya di lab dengan permission bank)
```

---

## Phase 4: POS Terminal Testing

```bash
# POS terminal — lebih umum di merchant

# Fingerprint POS terminal
# Vendor: Verifone, Ingenico, PAX, Castles, Telium

# Test 1: Default credentials pada admin menu
# Verifone: 1 ALPHA ALPHA 6 6831 (manager mode)
# Ingenico: 2913 (operator menu)

# Test 2: Debug port
# Banyak POS punya serial/USB debug port yang aktif
# Terhubung dengan USB-serial adapter
minicom -D /dev/ttyUSB0 -b 115200

# Test 3: Software version
# Catat versi → cari CVE

# Test 4: Network traffic
# Apakah kartu data terenkripsi sebelum dikirim?
# Point-to-Point Encryption (P2PE) seharusnya encrypt dari swipe
```

**Prompt Claude Code:**
```
POS terminal Verifone VX520 di merchant test:
- Admin menu berhasil diakses dengan default PIN
- Firmware version: 32xxxx.24 (outdated)
- USB debug port aktif (terhubung dengan minicom)
- Debug output menampilkan raw transaction data

Dari traffic capture saat transaksi test:
- Data terkirim via HTTPS ke payment gateway
- Sebelum HTTPS: card data terlihat di memory dump (tidak ter-P2PE)
- PAN (nomor kartu) muncul di log file: /var/log/pos.log

Analisis:
1. PAX data di log file — PCI-DSS violation apa?
2. Default PIN berhasil — risk assessment?
3. Tidak ada P2PE — jika memory scraped, data kartu bisa dicuri?
4. Rekomendasi: P2PE implementation, disable debug port, update firmware
```

---

## Checklist ATM & POS Pentest

```
[ ] Physical security inspection (skimmer, kamera tersembunyi)
[ ] Network port scan ATM/POS
[ ] OS version dan patch level (Windows EOL?)
[ ] RDP/remote management protection
[ ] Default credential testing
[ ] Encryption in transit (HTTPS? P2PE?)
[ ] Log file audit (PAN/CVV tersimpan di log? PCI-DSS violation!)
[ ] Network segmentation (ATM network terisolasi dari network lain?)
[ ] HSM (Hardware Security Module) — apakah PIN diproses di HSM?
[ ] Laporan: kaitkan setiap temuan dengan PCI-DSS requirement
```

---

## Durasi Estimasi

| Sub-skenario | Durasi |
|-------------|-----------|----------|
| ATM physical + network assessment | 3–5 hari | 7–10 hari |
| POS terminal security review | 2–3 hari | 4–6 hari |
