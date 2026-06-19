# Tutorial Penetration Testing dengan Claude Code di VSCode

## Tutorial Dasar

| File | Konten |
|------|--------|
| [01-pentest-dengan-claude-code.md](01-pentest-dengan-claude-code.md) | Tutorial lengkap 5 fase pentest dengan Claude Code |
| [02-prompt-templates-pentest.md](02-prompt-templates-pentest.md) | 30+ prompt siap pakai untuk pentest |
| [03-ctf-workflow.md](03-ctf-workflow.md) | Workflow CTF per kategori (web, crypto, forensics, pwn, rev) |

---

## Tutorial Skenario — Matriks Lengkap

Lihat [scenarios/00-overview-dan-rekomendasi.md](scenarios/00-overview-dan-rekomendasi.md) untuk penjelasan dan rekomendasi urutan belajar.

### Skenario Utama (Akses × Metode)

| Skenario | Black Box | Grey Box | White Box |
|----------|-----------|----------|-----------|
| **Lokal** `10.X.X.X` | [01](scenarios/01-lokal-blackbox.md) | [02](scenarios/02-lokal-greybox.md) | [03](scenarios/03-lokal-whitebox.md) |
| **Internal** `210.57.X.X` | [04](scenarios/04-internal-blackbox.md) | [05](scenarios/05-internal-greybox.md) | [06](scenarios/06-internal-whitebox.md) |
| **Internet Publik** | [07](scenarios/07-internet-blackbox.md) | [08](scenarios/08-internet-greybox.md) | [09](scenarios/09-internet-whitebox.md) |

### Skenario Tambahan (Berdasarkan Tipe Aplikasi)

| # | Skenario | File | Berlaku Untuk |
|---|----------|------|---------------|
| 10 | Aplikasi CMS (WordPress/Joomla/Drupal) | [10](scenarios/10-cms-wordpress.md) | Semua environment |
| 11 | SPA + REST/GraphQL API | [11](scenarios/11-spa-api.md) | Semua environment |
| 12 | E-Commerce + Payment Gateway | [12](scenarios/12-ecommerce.md) | Internet/Internal |
| 13 | Enterprise SSO / OAuth 2.0 | [13](scenarios/13-sso-oauth.md) | Internal/Internet |
| 14 | Mobile Backend / API-First App | [14](scenarios/14-mobile-backend.md) | Internet/Internal |

**Total: 14 tutorial skenario**

---

### Referensi Pendukung

| File | Konten |
|------|--------|
| [scenarios/15-durasi-dan-jadwal.md](scenarios/15-durasi-dan-jadwal.md) | Estimasi durasi + jadwal harian detail (dengan bantuan AI) |
| [scenarios/15b-durasi-tanpa-ai.md](scenarios/15b-durasi-tanpa-ai.md) | Estimasi durasi + jadwal yang disesuaikan tanpa bantuan AI |
| [scenarios/16-daftar-tools.md](scenarios/16-daftar-tools.md) | Daftar lengkap tools, instalasi, dan matriks tools per skenario |

---

---

## Network Penetration Testing

Lihat [network/00-overview-network.md](network/00-overview-network.md) untuk overview dan perbandingan dengan application pentest.

| # | Skenario | File |
|---|----------|------|
| 1 | Internal LAN (Jaringan Korporat + Active Directory) | [network/01-internal-lan.md](network/01-internal-lan.md) |
| 2 | Wireless / WiFi (WPA cracking, Evil Twin, VLAN) | [network/02-wireless-wifi.md](network/02-wireless-wifi.md) |
| 3 | VPN & Remote Access | [network/03-vpn-remote-access.md](network/03-vpn-remote-access.md) |
| 4 | Segmentasi Jaringan & Firewall | [network/04-segmentasi-firewall.md](network/04-segmentasi-firewall.md) |
| 5 | DMZ (Demilitarized Zone) | [network/05-dmz.md](network/05-dmz.md) |
| 6 | Cloud Network (AWS/GCP/Azure) | [network/06-cloud-network.md](network/06-cloud-network.md) |
| 7 | OT/ICS/SCADA (Industrial Network) | [network/07-ot-ics-scada.md](network/07-ot-ics-scada.md) |

---

## Hardware Penetration Testing

Lihat [hardware/00-overview-hardware.md](hardware/00-overview-hardware.md) untuk overview dan peralatan yang diperlukan.

| # | Skenario | File |
|---|----------|------|
| 1 | IoT Devices (kamera, sensor, router IoT) | [hardware/01-iot-devices.md](hardware/01-iot-devices.md) |
| 2 | RFID & NFC (kartu akses gedung) | [hardware/02-rfid-nfc.md](hardware/02-rfid-nfc.md) |
| 3 | Network Devices (router/switch/firewall) | [hardware/03-network-devices.md](hardware/03-network-devices.md) |
| 4 | ATM & POS Terminals | [hardware/04-atm-pos.md](hardware/04-atm-pos.md) |
| 5 | Firmware & Embedded Systems | [hardware/05-firmware-embedded.md](hardware/05-firmware-embedded.md) |
| 6 | USB & Physical Ports | [hardware/06-usb-physical-ports.md](hardware/06-usb-physical-ports.md) |

---

## Personnel Penetration Testing (Social Engineering)

Lihat [personnel/00-overview-personnel.md](personnel/00-overview-personnel.md) untuk framework dan aturan etika yang wajib dipenuhi.

| # | Skenario | File |
|---|----------|------|
| 1 | Phishing Email Campaign | [personnel/01-phishing-email.md](personnel/01-phishing-email.md) |
| 2 | Vishing (Voice Phishing) | [personnel/02-vishing.md](personnel/02-vishing.md) |
| 3 | Smishing (SMS Phishing) | [personnel/03-smishing.md](personnel/03-smishing.md) |
| 4 | Physical Intrusion & Tailgating | [personnel/04-physical-intrusion.md](personnel/04-physical-intrusion.md) |
| 5 | USB Drop Attack | [personnel/05-usb-drop.md](personnel/05-usb-drop.md) |
| 6 | Pretexting & Impersonation | [personnel/06-pretexting-impersonation.md](personnel/06-pretexting-impersonation.md) |
| 7 | Baiting & Quid Pro Quo | [personnel/07-baiting-quidproquo.md](personnel/07-baiting-quidproquo.md) |

---

## Prasyarat

- VSCode dengan Claude Code terinstal
- Python 3.8+, tools: nmap, gobuster, nikto, sqlmap, wpscan
- Target yang **diotorisasi** atau platform lab (TryHackMe, HackTheBox)

---

> **Legal Notice:** Semua teknik hanya untuk digunakan pada sistem yang Anda miliki atau memiliki izin eksplisit tertulis. Penggunaan tanpa izin adalah ilegal dan melanggar UU ITE Indonesia.
