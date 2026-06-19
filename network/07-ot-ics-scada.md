# Network Pentest: OT/ICS/SCADA

**Target:** Operational Technology — sistem kontrol industri, SCADA, PLC, HMI  
**Contoh:** Sistem listrik, pabrik, water treatment, oil & gas  
**PERINGATAN KERAS:** Kesalahan pada OT network bisa menyebabkan kerusakan fisik nyata

---

## ⚠️ Peringatan Khusus OT/ICS

```
BACA INI SEBELUM MEMULAI:

1. OT/ICS bukan IT biasa — kesalahan bisa sebabkan:
   - Kerusakan mesin fisik (tidak bisa di-undo)
   - Bahaya keselamatan jiwa manusia
   - Gangguan layanan kritis (listrik, air, gas)

2. SELALU lakukan dalam jendela maintenance window
3. SELALU ada engineer OT yang standby selama testing
4. JANGAN pernah send payload berbahaya ke PLC/RTU nyata
5. Gunakan PASSIVE scanning sebelum active — atau simulasi di lab

6. Regulasi: UU No. 11/2008 (ITE), PP 71/2019 (SPSE),
   bisa bersinggungan dengan BSSN untuk infrastruktur kritis
```

---

## Arsitektur Umum OT/ICS

```
[Internet / IT Network]
    ↓
[IT/OT Boundary (Firewall/DMZ)]
    ↓
[Level 3 — Manufacturing IT: Historian, MES]
    ↓
[Level 2 — Supervisory: SCADA, HMI]
    ↓
[Level 1 — Control: PLC, DCS, RTU]
    ↓
[Level 0 — Field Devices: Sensor, Actuator, Motor]
```

---

## Phase 1: Passive OT Network Discovery

```bash
# SELALU mulai dengan passive scanning — tidak mengirim packet ke device

# Wireshark — tangkap traffic OT
sudo tcpdump -i eth0 -w ot_traffic.pcap

# Analisis traffic — identifikasi protokol OT
tshark -r ot_traffic.pcap -Y "modbus || dnp3 || s7comm || enip || bacnet" \
  -T fields -e ip.src -e ip.dst -e frame.protocols

# Identifikasi OT protocols dari pcap:
# - Modbus/TCP (port 502) — paling umum
# - DNP3 (port 20000) — utility/energy
# - EtherNet/IP (port 44818) — Allen Bradley PLC
# - S7Comm (port 102) — Siemens PLC
# - BACnet (port 47808) — Building automation
# - Profinet, OPC-UA, IEC 61850
```

---

## Phase 2: OT Asset Discovery (Hati-hati)

```bash
# Redpoint Nmap scripts — OT-specific discovery
# passive-safe, BACA DOKUMENTASI sebelum run
nmap --script enip-info -p 44818 OT_subnet/24   # Allen Bradley
nmap --script s7-info -p 102 OT_subnet/24        # Siemens
nmap --script modbus-discover -p 502 OT_subnet/24 # Modbus

# Shodan — jika OT terekspos internet (sangat berbahaya)
shodan search "port:502 Modbus"          # Modbus di internet!
shodan search "port:102 Siemens S7"      # Siemens PLC
shodan search "port:44818 Allen-Bradley"
```

**Prompt Claude Code:**
```
OT network discovery dari traffic capture (passive):

Protokol yang terdeteksi:
- Modbus/TCP (port 502): 8 PLC, polling setiap 1 detik
- EtherNet/IP (port 44818): 3 Allen-Bradley PLC
- OPC-UA (port 4840): 1 historian server
- HTTP (port 80): 2 HMI (web-based!)

Dari Shodan (informasi tambahan):
- 1 Siemens S7-300 PLC terekspos di internet! (port 102)

Buat risk assessment:
1. Apa risiko PLC yang terekspos internet?
2. Modbus tanpa autentikasi — apa yang bisa dilakukan penyerang?
3. HMI web-based — attack surface apa?
4. Urutan prioritas dari risiko tertinggi
5. Rekomendasi arsitektur yang aman (Purdue Model)
```

---

## Phase 3: Modbus Protocol Testing

```bash
# Modbus tidak memiliki autentikasi sama sekali
# Apapun yang terkoneksi ke port 502 bisa baca/tulis coil/register

# modbuscli — Modbus CLI client
pip install modbus_cli

# Baca holding registers (status sensor, setpoint)
modbus read --host 192.168.10.5 --port 502 holding_registers 0 10

# Baca coil status (ON/OFF status)
modbus read --host 192.168.10.5 --port 502 coils 0 8

# ⚠️ JANGAN LAKUKAN INI DI SISTEM NYATA TANPA SANGAT YAKIN:
# Tulis coil (bisa matikan perangkat!)
# modbus write --host 192.168.10.5 coils 0 1

# PyModbus untuk scripting
from pymodbus.client import ModbusTcpClient
client = ModbusTcpClient('192.168.10.5', port=502)
client.connect()
result = client.read_holding_registers(0, 10, slave=1)
print(f"Register values: {result.registers}")
client.close()
```

**Prompt Claude Code:**
```
Modbus testing pada PLC di OT network (authorized, dalam maintenance window).

Read holding registers dari PLC (slave ID 1):
Register 0: 4523  (kemungkinan: temperature 45.23°C?)
Register 1: 8000  (kemungkinan: pressure 80.00 bar?)
Register 2: 1     (kemungkinan: pump ON)
Register 3: 0     (kemungkinan: valve CLOSED)

Analisis:
1. Bagaimana interpret nilai register ini tanpa dokumentasi PLC?
2. Apa yang bisa dilakukan penyerang dengan akses write ke register ini?
3. Apakah Modbus ini seharusnya di-isolate dari jaringan lain?
4. Rekomendasi: autentikasi, monitoring anomali Modbus
```

---

## Phase 4: HMI Web Interface Testing

HMI berbasis web sering memiliki kerentanan aplikasi web standar:

```bash
# HMI biasanya punya web interface untuk monitoring
curl -I http://hmi.ot.internal

# Test default credentials yang umum untuk HMI:
# Siemens WinCC: Administrator/(kosong), admin/admin
# Wonderware: administrator/wonderware
# Ignition: admin/password
# GE iFIX: (tanpa auth default)
# Rockwell FactoryTalk: Administrator/(kosong)

# Test directory listing
curl http://hmi.ot.internal/
# Cari: project files, tag database, historian data
```

**Prompt Claude Code:**
```
HMI Siemens WinCC di OT network:
- URL: http://192.168.10.200/awp/
- Default credential berhasil: Administrator/(blank password)
- Berhasil login ke panel monitoring

Yang terlihat setelah login:
- Real-time dashboard semua sensor dan aktuator
- Bisa control: start/stop pump, buka/tutup valve, set setpoint
- Log history 30 hari
- User management: hanya 1 user (Administrator)

Buat risk assessment:
1. Apa dampak jika penyerang bisa akses HMI ini?
2. Control apa yang paling berbahaya jika disalahgunakan?
3. Apakah ini termasuk infrastruktur kritis? Regulasi apa yang berlaku di Indonesia?
4. Rekomendasi: hardening HMI tanpa mengganggu operasi
```

---

## Phase 5: IT/OT Boundary Testing

```bash
# Dari IT network, test apakah bisa reach OT
ping 192.168.10.5       # PLC
nc -zv 192.168.10.5 502 # Modbus

# Seharusnya BLOCKED sepenuhnya
# Jika tidak — segmentasi IT/OT gagal

# Cek historian server (sering menjadi jembatan IT-OT)
# Historian biasanya di DMZ OT — bisa reach kedua sisi
nmap -p 1433,5432,3306 historian.ot.internal  # database historian
```

---

## Checklist OT/ICS/SCADA Pentest

```
[ ] Pastikan maintenance window dan OT engineer standby
[ ] Passive traffic capture & protocol identification
[ ] Asset discovery (passive/active dengan sangat hati-hati)
[ ] Cek OT devices yang terekspos internet (Shodan)
[ ] Modbus/DNP3/S7 enumeration (READ ONLY)
[ ] HMI default credential testing
[ ] HMI web interface security testing
[ ] IT/OT boundary test (apakah benar-benar terpisah?)
[ ] Historian server security
[ ] Remote access (VPN ke OT, jump server) audit
[ ] Laporan: kaitkan dengan IEC 62443, NIST SP 800-82
[ ] JANGAN lakukan write ke PLC tanpa konfirmasi OT engineer
```

---

## Durasi Estimasi

| Sub-skenario | Durasi |
|-------------|-----------|----------|
| Passive OT assessment | 2–3 hari | 4–6 hari |
| Full OT pentest (dengan maintenance window) | 5–8 hari | 12–18 hari |

*Catatan: OT pentest memerlukan koordinasi jauh lebih banyak dari IT pentest biasa*
