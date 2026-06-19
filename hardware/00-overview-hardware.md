# Hardware Penetration Testing — Overview

Hardware pentest menguji **keamanan perangkat fisik**: dari IoT, perangkat jaringan,
sistem embedded, hingga perangkat yang menyimpan atau memproses data sensitif.

---

## Skenario yang Tersedia

| # | Skenario | File | Fokus Utama |
|---|----------|------|-------------|
| 1 | IoT Devices | [01-iot-devices.md](01-iot-devices.md) | Firmware analysis, default creds, exposed service |
| 2 | RFID & NFC (Access Control) | [02-rfid-nfc.md](02-rfid-nfc.md) | Cloning kartu, replay attack, bypass pintu |
| 3 | Network Devices (Router/Switch/Firewall) | [03-network-devices.md](03-network-devices.md) | Default creds, firmware vuln, config extraction |
| 4 | ATM & POS Terminals | [04-atm-pos.md](04-atm-pos.md) | Skimming, black-box attack, OS vulnerability |
| 5 | Firmware & Embedded Systems | [05-firmware-embedded.md](05-firmware-embedded.md) | Firmware extraction, binwalk, reverse engineering |
| 6 | USB & Physical Ports | [06-usb-physical-ports.md](06-usb-physical-ports.md) | USB attack, debug port, JTAG/UART |

---

## Mengapa Hardware Pentest Penting

- **Perangkat IoT** sering memiliki keamanan minimal dan menjadi pintu masuk ke jaringan
- **Default credentials** pada perangkat jaringan masih sangat umum ditemukan
- **Firmware** sering mengandung hardcoded secrets yang tidak bisa di-patch dari jauh
- **Physical access** ke port debug bisa bypass seluruh keamanan software
- **RFID/NFC** yang lemah memungkinkan bypass akses fisik gedung

---

## Perbedaan Hardware vs Application/Network Pentest

| Aspek | Application | Network | Hardware |
|-------|-------------|---------|---------|
| Target | Software/API | Infrastruktur | Perangkat fisik |
| Akses | Remote | Remote/Internal | Perlu akses fisik |
| Tools | Burp, sqlmap | nmap, Responder | Flasher, logic analyzer, Proxmark |
| Skill tambahan | Web security | Networking | Elektronik, soldering, RE |
| Risk | Data breach | Network compromise | Akses fisik tak terbatas |

---

## Tools Utama Hardware Pentest

| Kategori | Tools | Fungsi |
|----------|-------|--------|
| Firmware Analysis | Binwalk, Firmwalker, FACT | Extract & analisis firmware |
| RFID/NFC | Proxmark3, ACR122U, Flipper Zero | Clone/emulate kartu |
| USB Attack | USB Rubber Ducky, BadUSB, P4wnP1 | Keystroke injection |
| Debug Interface | Bus Pirate, JTAGulator, UART adapter | Akses debug port |
| Traffic Analysis | Wireshark, Logic Analyzer | Analisis komunikasi hardware |
| Reverse Engineering | Ghidra, Binwalk, radare2 | Analisis firmware/binary |
| Network Device | RouterSploit, Routerhunter | Exploit router/switch |

---

## Persiapan Khusus Hardware Pentest

```
Peralatan fisik yang mungkin diperlukan:
- Proxmark3 atau Flipper Zero (RFID/NFC)
- USB Rubber Ducky atau O.MG Cable (USB attack)
- UART/JTAG adapter (debug port)
- Soldering iron (jika perlu akses ke board)
- Logic analyzer (analisis sinyal digital)
- Multimeter
- Anti-static equipment
```

---

## Disclaimer Khusus Hardware

> Hardware pentest memerlukan **izin eksplisit** untuk menyentuh perangkat fisik.
> Beberapa teknik (soldering, membuka casing) bersifat **destruktif dan tidak reversible**.
> Selalu dokumentasikan kondisi perangkat sebelum dan sesudah pengujian.
