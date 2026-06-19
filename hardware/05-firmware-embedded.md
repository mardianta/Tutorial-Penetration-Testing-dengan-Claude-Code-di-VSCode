# Hardware Pentest: Firmware & Embedded Systems

**Target:** Firmware device apapun — router, kamera, IoT, printer, industrial device  
**Durasi:** 3–7 hari per device  
**Tools:** Binwalk, Ghidra, radare2, UART adapter, JTAG

---

## Phase 1: Firmware Acquisition

Ada beberapa cara mendapatkan firmware:

```bash
# Cara 1: Download dari vendor website (paling mudah)
# Cari di: support.vendor.com/downloads/model-number

# Cara 2: Dari web interface device (jika sudah login)
curl -u admin:admin \
  http://192.168.1.1/firmware.bin \
  -o firmware_original.bin

# Cara 3: Intercept update process
# Setup MITM, device saat update → capture firmware dari HTTP/HTTPS
mitmproxy --ssl-insecure -w firmware_capture.mitm

# Cara 4: Flash chip dumping (hardware method)
# Perlu akses fisik ke board
# Identifikasi flash chip: SPI flash, NAND flash
# Gunakan: CH341A, Bus Pirate, atau flashrom
sudo flashrom -p ch341a_spi -r firmware_dump.bin

# Cara 5: UART/serial console
# Akses root shell via serial port → dd if=/dev/mtd0 > /tmp/firmware.bin
```

**Prompt Claude Code:**
```
Saya berhasil mendapatkan firmware router TP-Link TL-WR841N v14.
Ukuran file: 1.7MB
Output 'file' command: data

Langkah analisis firmware:
1. Cara identifikasi format dan struktur firmware
2. Binwalk command sequence yang optimal
3. Apa yang dicari setelah berhasil extract file system?
4. Cara identifikasi apakah firmware terenkripsi atau terkompresi
```

---

## Phase 2: Firmware Analysis dengan Binwalk

```bash
# Step 1: Entropy analysis — cari area terenkripsi
binwalk -E firmware.bin
# Area dengan entropy tinggi (mendekati 1.0) = terenkripsi/terkompresi

# Step 2: Signature scan
binwalk firmware.bin
# Identifikasi: bootloader, kernel, squashfs filesystem, config

# Step 3: Extract
binwalk -e firmware.bin          # otomatis extract
binwalk -Me firmware.bin         # recursive extract (matryoshka)
# Hasil: folder _firmware.bin.extracted/

# Step 4: Analisis file system
ls _firmware.bin.extracted/squashfs-root/
find . -type f -name "*.conf" -o -name "*.sh" -o -name "*.cgi"

# Step 5: Cari credentials dan secrets
grep -r "password\|passwd\|secret\|api_key\|token" . \
  --include="*.conf" --include="*.sh" --include="*.lua" \
  --include="*.py" --include="*.json"

# Step 6: Cari binary setuid
find . -perm -4000 -type f

# Step 7: Strings pada binary penting
strings ./usr/bin/httpd | grep -i "pass\|secret\|admin\|key"
```

**Prompt Claude Code:**
```
Binwalk berhasil extract firmware TP-Link TL-WR841N.

File system: squashfs-root/
Temuan awal:
1. /etc/passwd: root:x, admin:x (tidak ada hash — shadow?)
2. /etc/shadow: root:$1$GTN.gpA/$TZ (MD5 hash!)
3. /usr/bin/httpd: binary web server (GoAhead)
4. /etc/config/: folder konfigurasi

Dari strings pada /usr/bin/httpd:
"ThomsonR99" ditemukan ← hardcoded?
"Authorization: Basic" ← HTTP basic auth
"admin" ditemukan banyak kali

Dari /etc/config/system:
config system
    option hostname 'OpenWrt'
    option timezone 'UTC'

Analisis:
1. MD5 hash di shadow — cara crack?
2. "ThomsonR99" — apakah ini hardcoded credential?
3. GoAhead web server — CVE yang dikenal?
4. Langkah analisis lebih lanjut yang disarankan
```

---

## Phase 3: Emulasi Firmware (QEMU)

```bash
# Emulasi firmware tanpa hardware fisik
# Berguna untuk dynamic analysis

# Install QEMU
sudo apt install qemu-user-static

# Identifikasi arsitektur CPU dari firmware
binwalk -A firmware.bin | grep "CPU"
file _extracted/squashfs-root/bin/busybox
# ELF 32-bit LSB executable, MIPS

# Copy library yang diperlukan
cp /usr/bin/qemu-mipsel-static _extracted/squashfs-root/usr/bin/

# Chroot ke firmware
sudo chroot _extracted/squashfs-root/ /bin/sh
# Sekarang berada di dalam environment firmware!

# Jalankan web server firmware
/usr/bin/httpd -f -h /www

# Tools untuk emulasi penuh: FIRMAE
git clone https://github.com/pr0v3rbs/FirmAE
cd FirmAE && ./run.sh -r TPLINK firmware.bin
```

---

## Phase 4: Debug Interface (UART/JTAG)

```bash
# UART — Serial console yang sering ada di board PCB

# Step 1: Buka casing device (setelah izin dan dokumentasi kondisi)
# Step 2: Identifikasi UART pins (GND, TX, RX, VCC)
#         Gunakan multimeter: GND=0V, VCC=3.3V/5V, TX=oscillating

# Step 3: Sambungkan dengan USB-UART adapter
# CH340, FT232, atau CP2102
# TX device → RX adapter, RX device → TX adapter, GND → GND

# Step 4: Cari baud rate (biasanya 115200)
minicom -D /dev/ttyUSB0 -b 115200
# Atau gunakan baudrate scanner

# Jika berhasil — dapat akses boot console
# Tekan tombol untuk interrupt boot dan masuk ke bootloader (U-Boot)
# Di U-Boot: bisa dump firmware, bypass password

# JTAG — debug interface lebih powerful
# Tools: OpenOCD, JTAGulator
# Bisa: halting CPU, memory dump, bypass security
openocd -f interface/jlink.cfg -f target/at91.cfg
```

**Prompt Claude Code:**
```
Berhasil connect ke UART console router.
Boot log menampilkan U-Boot kemudian Linux booting.

Setelah boot selesai, muncul login prompt:
"tplink login: "

Dari boot log tercatat:
- U-Boot 2010.06 (Mar 15 2019)
- Linux 3.14.77
- mtd0: 0x00000000-0x00020000: "u-boot"
- mtd3: 0x000e0000-0x000f0000: "config"
- Flash map terlihat jelas

Tidak tahu password login.

Buatkan:
1. Cara interrupt U-Boot untuk masuk bootloader mode
2. Di U-Boot: cara dump /etc/shadow untuk offline crack
3. Cara boot dengan init=/bin/sh untuk bypass password
4. Cara mount flash dan extract konfigurasi dari U-Boot
```

---

## Phase 5: Reverse Engineering Binary

```bash
# Analisis binary yang tidak ada source code-nya

# Ghidra — free dari NSA
# Import binary → auto-analysis → decompile ke C

# radare2 — CLI reverse engineering
r2 /path/to/binary
[0x00000000]> aaa    # analyze all
[0x00000000]> afl    # list all functions
[0x00000000]> pdf @ main  # disassemble main function
[0x00000000]> pdg @ main  # decompile main (dengan r2ghidra plugin)

# strings — cepat cari credential hardcoded
strings binary | grep -iE "pass|admin|key|secret|token"

# ltrace/strace — lihat function calls saat runtime (di emulasi)
strace ./httpd 2>&1 | grep "open\|read\|write"
```

---

## Checklist Firmware & Embedded Pentest

```
[ ] Firmware acquisition (web interface, vendor site, flash dump)
[ ] Entropy analysis (terenkripsi atau tidak)
[ ] Binwalk extraction dan file system exploration
[ ] Credential hunting (grep secrets di semua file)
[ ] Shadow file extraction dan crack
[ ] Binary analysis untuk hardcoded secrets
[ ] QEMU emulasi (jika memungkinkan)
[ ] UART/JTAG access (jika akses fisik diizinkan)
[ ] Dynamic testing setelah emulasi
[ ] Laporan: hardcoded creds, outdated OS/library, debug interface aktif
```

---

## Durasi Estimasi

| Sub-skenario | Durasi |
|-------------|-----------|----------|
| Firmware static analysis saja | 2–3 hari | 5–7 hari |
| Firmware + emulasi + RE | 4–6 hari | 10–15 hari |
| Firmware + hardware (UART/JTAG) | 5–7 hari | 12–18 hari |
