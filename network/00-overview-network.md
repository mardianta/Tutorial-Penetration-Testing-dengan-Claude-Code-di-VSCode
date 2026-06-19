# Network Penetration Testing — Overview

Network pentest berfokus pada **infrastruktur jaringan**: bagaimana data bergerak,
siapa yang bisa mengaksesnya, dan apakah segmentasi & kontrol akses benar-benar bekerja.

---

## Skenario yang Tersedia

| # | Skenario | File | Fokus Utama |
|---|----------|------|-------------|
| 1 | Internal LAN (Jaringan Korporat) | [01-internal-lan.md](01-internal-lan.md) | Recon jaringan, lateral movement, AD |
| 2 | Wireless / WiFi | [02-wireless-wifi.md](02-wireless-wifi.md) | WPA cracking, rogue AP, evil twin |
| 3 | VPN & Remote Access | [03-vpn-remote-access.md](03-vpn-remote-access.md) | Credential attack, split tunneling, MFA bypass |
| 4 | Segmentasi Jaringan & Firewall | [04-segmentasi-firewall.md](04-segmentasi-firewall.md) | VLAN hopping, firewall rule bypass |
| 5 | DMZ (Demilitarized Zone) | [05-dmz.md](05-dmz.md) | Pivot dari DMZ ke internal |
| 6 | Cloud Network (AWS/GCP/Azure) | [06-cloud-network.md](06-cloud-network.md) | Security group, VPC, metadata endpoint |
| 7 | OT/ICS/SCADA | [07-ot-ics-scada.md](07-ot-ics-scada.md) | Industrial network, Modbus, legacy protocol |

---

## Perbedaan Network vs Application Pentest

| Aspek | Application Pentest | Network Pentest |
|-------|-------------------|-----------------|
| Target | Web app, API, mobile | Router, switch, firewall, protokol |
| Tools | Burp Suite, sqlmap, nmap | Wireshark, Metasploit, Impacket, Responder |
| Kerentanan | SQLi, XSS, IDOR | MITM, lateral movement, credential relay |
| Impact | Data breach, akun terkompromi | Akses seluruh jaringan, pivot ke semua sistem |
| Skill | Web security | Networking + OS + protokol |

---

## Metodologi Umum Network Pentest

```
1. Network Discovery     → temukan semua host, subnet, dan service
2. Enumeration           → identifikasi OS, versi, konfigurasi
3. Vulnerability Scan    → cari kerentanan di setiap host/service
4. Exploitation          → exploit kerentanan untuk akses
5. Post-Exploitation     → lateral movement, privilege escalation
6. Pivoting              → akses jaringan tersegmentasi dari dalam
7. Reporting             → dokumentasi semua temuan + rekomendasi
```

---

## Tools Utama Network Pentest

| Kategori | Tools |
|----------|-------|
| Discovery | nmap, masscan, netdiscover, arp-scan |
| Enumeration | nmap scripts, enum4linux, smbclient, ldapdomaindump |
| MITM | Responder, Ettercap, Bettercap, arpspoof |
| Credential | Hydra, CrackMapExec, Impacket |
| Wireless | Aircrack-ng, Kismet, Hostapd-wpe, Wifite |
| Exploitation | Metasploit, Impacket suite |
| Traffic Analysis | Wireshark, tcpdump, tshark |
| Pivoting | Chisel, sshuttle, proxychains |

---

## Regulasi & Standar Relevan

| Standar | Relevansi |
|---------|-----------|
| ISO 27001 | Network security controls |
| NIST SP 800-115 | Technical guide to network pentest |
| CIS Benchmarks | Hardening guide per device |
| PCI-DSS | Network segmentation requirement |
| Permenkominfo 4/2016 | Keamanan jaringan sistem elektronik Indonesia |
