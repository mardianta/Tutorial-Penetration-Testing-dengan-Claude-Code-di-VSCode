# Hardware Pentest: USB & Physical Port Security

**Target:** Port USB, port serial, port debug pada komputer dan device  
**Durasi:** 1–2 hari  
**Tools:** USB Rubber Ducky, O.MG Cable, BadUSB, Bus Pirate

---

## Phase 1: USB Attack Assessment

```bash
# Test 1: Apakah komputer target bisa autorun USB?
# Windows: autorun sudah disabled default sejak Win7
# Tapi: HID (Human Interface Device) tetap berjalan!

# USB Rubber Ducky — emulasi keyboard, inject keystrokes
# Saat dicolokan: dianggap sebagai keyboard USB yang valid
# Ketik payload otomatis dalam milidetik

# Payload contoh (untuk lab/test):
# payload.txt
DELAY 1000
GUI r                    # Windows key + R (Run dialog)
DELAY 500
STRING powershell -WindowStyle hidden -EncodedCommand <base64>
ENTER
```

**Prompt Claude Code:**
```
Saya akan test USB security di workstation target (authorized).
OS: Windows 10 Pro

Buatkan payload USB Rubber Ducky yang:
1. Membuka PowerShell tersembunyi
2. Download dan jalankan script dari server test saya (10.10.10.5)
3. Kirim output (hostname, username, IP) ke server test
4. Selesai dalam < 10 detik sebelum user sadar

Ini untuk menguji apakah USB attack berhasil di lingkungan ini.
Juga: rekomendasi pencegahan apa yang efektif?
```

---

## Phase 2: BadUSB Assessment

```bash
# BadUSB — firmware USB device dimodifikasi
# Bahkan USB biasa bisa dijadikan BadUSB jika firmwarenya diflash

# P4wnP1 — Raspberry Pi Zero sebagai USB attack platform
# Bisa jadi: keyboard, network adapter, storage sekaligus
# Setup:
# 1. Flash P4wnP1 ke Raspberry Pi Zero W
# 2. Konfigurasi payload
# 3. Colokkan — otomatis eksekusi

# O.MG Cable — kabel USB yang kelihatan normal tapi ada implant di dalam
# Bisa: inject keystrokes, WiFi hotspot tersembunyi

# Test: apakah endpoint protection memblokir USB HID baru?
# Windows Defender: tidak memblokir HID secara default
# Solusi: USB device control via GPO atau endpoint security (Symantec, CrowdStrike)
```

---

## Phase 3: USB Data Exfiltration Test

```bash
# Test: apakah data bisa di-copy ke USB?
# Di banyak organisasi, USB storage seharusnya diblokir via policy

# Windows — cek USB storage policy
# Registry: HKLM\SYSTEM\CurrentControlSet\Services\UsbStor
# Start = 4 → USB storage disabled
# Start = 3 → USB storage enabled

# Test langsung: colokan USB storage
# Jika muncul di Explorer → policy tidak di-enforce

# Teknik bypass USB block:
# 1. USB-to-Ethernet adapter (HID, bukan storage → tidak terblokir)
# 2. USB keyboard + HID → inject command untuk exfil via network
# 3. USB Rubber Ducky → copy file via PowerShell ke cloud
```

---

## Phase 4: Physical Port Assessment

```bash
# Selain USB — test semua port fisik yang ada

# 1. Thunderbolt / DMA Attack
# Thunderbolt mengizinkan DMA (Direct Memory Access)
# Tools: PCILeech — baca/tulis RAM langsung via Thunderbolt
sudo ./pcileech write -min 0x0 -max 0xffffffff -out memory.dump
# Bisa extract: password, encryption keys, credential dari RAM

# 2. FireWire — DMA attack (legacy)
# sudo fwtool -v (jika ada FireWire port)

# 3. SD Card slot
# Test apakah bisa boot dari SD card atau execute script

# 4. Kensington Lock
# Apakah laptop terkunci ke meja? (mencegah pencurian fisik)
```

---

## Phase 5: Rekomendasi & Pencegahan

**Prompt Claude Code:**
```
Hasil USB & physical port assessment:
- USB storage: TIDAK terblokir (bisa copy data ke flashdisk)
- USB HID: tidak ada whitelist (USB Rubber Ducky berhasil)
- Thunderbolt: aktif tanpa Thunderbolt security level
- Komputer tidak terkunci ke meja
- BIOS: tidak ada password BIOS
- Boot order: bisa boot dari USB

Buatkan rekomendasi hardening komprehensif:
1. Group Policy untuk blokir USB storage tapi izinkan keyboard/mouse
2. Thunderbolt security level yang tepat
3. BIOS password dan secure boot configuration
4. Physical security: kensington lock, port blocker
5. Endpoint DLP (Data Loss Prevention) solution
6. Security awareness training tentang USB attack
```

---

## Checklist USB & Physical Port Pentest

```
[ ] Test USB storage — apakah bisa copy data ke USB?
[ ] Test USB HID (Rubber Ducky) — apakah bisa inject keystrokes?
[ ] Test USB charging port — apakah hanya charge atau data juga?
[ ] Test Thunderbolt / DMA access
[ ] Test boot dari USB/SD card (BIOS check)
[ ] Cek password BIOS/UEFI
[ ] Cek secure boot status
[ ] Physical: apakah device terkunci ke meja?
[ ] Cek port physical yang tidak terpakai (tapi aktif)
[ ] Laporan: USB policy recommendation + endpoint solution
```

---

## Durasi Estimasi

| Sub-skenario | Durasi |
|-------------|-----------|----------|
| USB & port assessment (per workstation) | 0.5 hari | 1 hari |
| Full physical security audit (50 workstation) | 2–3 hari | 4–6 hari |
