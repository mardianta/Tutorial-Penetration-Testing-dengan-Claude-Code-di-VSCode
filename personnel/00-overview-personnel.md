# Personnel Penetration Testing (Social Engineering) — Overview

Personnel pentest menguji **keamanan manusia**: seberapa rentan karyawan terhadap
manipulasi psikologis yang bisa digunakan penyerang untuk bypass keamanan teknis.

> "Manusia adalah mata rantai terlemah dalam keamanan informasi."
> Teknologi secanggih apapun bisa ditembus jika satu karyawan terpancing.

---

## Skenario yang Tersedia

| # | Skenario | File | Vektor Serangan |
|---|----------|------|----------------|
| 1 | Phishing Email | [01-phishing-email.md](01-phishing-email.md) | Email palsu berisi link/attachment berbahaya |
| 2 | Vishing (Voice Phishing) | [02-vishing.md](02-vishing.md) | Telepon palsu mengaku IT/manajemen/vendor |
| 3 | Smishing (SMS Phishing) | [03-smishing.md](03-smishing.md) | SMS palsu dengan link berbahaya |
| 4 | Physical Intrusion & Tailgating | [04-physical-intrusion.md](04-physical-intrusion.md) | Masuk area terlarang secara fisik |
| 5 | USB Drop Attack | [05-usb-drop.md](05-usb-drop.md) | USB berbahaya yang sengaja "ditemukan" |
| 6 | Pretexting & Impersonation | [06-pretexting-impersonation.md](06-pretexting-impersonation.md) | Berpura-pura jadi orang/pihak lain |
| 7 | Baiting & Quid Pro Quo | [07-baiting-quidproquo.md](07-baiting-quidproquo.md) | Iming-iming hadiah/pertukaran layanan |

---

## Mengapa Personnel Pentest Penting

```
Fakta dari laporan Verizon DBIR:
- 82% breach melibatkan faktor manusia
- 36% breach melibatkan phishing
- Social engineering adalah vektor #1 untuk initial access
```

Perusahaan bisa memiliki firewall terbaik, WAF, EDR — tapi jika satu karyawan
klik link phishing dan memasukkan credentials, semua pertahanan teknis bypass.

---

## Perbedaan Personnel vs Pentest Lainnya

| Aspek | Technical Pentest | Personnel Pentest |
|-------|------------------|------------------|
| Target | Sistem, software, hardware | Manusia / perilaku |
| Tools | Software, exploit | Psikologi, rekayasa sosial |
| Skill | Teknis | Komunikasi, acting, riset |
| Metrik sukses | Akses sistem | Karyawan termakan pancingan |
| Deliverable | CVE, PoC | Tingkat klik, laporan perilaku |
| Etika | Sangat ketat | **Sangat sangat ketat** |

---

## Framework & Metodologi

| Framework | Deskripsi |
|-----------|-----------|
| **SEEF** (Social Engineering Execution Framework) | Standar pelaksanaan |
| **MITRE ATT&CK** — Initial Access tactic | Kategorisasi teknik SE |
| **SET** (Social Engineering Toolkit) | Tools untuk simulasi |
| **GoPhish** | Platform phishing campaign |

---

## Aturan Etika yang Wajib Dipenuhi

```
SEBELUM:
✓ Written authorization dengan scope yang sangat jelas
✓ Tentukan: departemen/individu mana yang boleh ditarget
✓ Get-out-of-jail letter untuk tester (jika ditangkap satpam)
✓ Konfirmasi: boleh/tidak menggunakan brand perusahaan sebagai pretext?
✓ Pastikan: HR dan legal sudah tahu (tapi bukan target karyawan)

SELAMA:
✓ Tidak boleh menyebabkan kerusakan psikologis pada target
✓ Tidak mengumpulkan PII di luar yang diperlukan untuk PoC
✓ Stop jika target menunjukkan tanda stres berlebihan
✓ Tidak boleh menggunakan ancaman atau tekanan berlebihan

SETELAH:
✓ Debrief kepada karyawan yang "jatuh" — edukasi, BUKAN menghukum
✓ Laporan bersifat anonim (tidak sebut nama individu)
✓ Fokus pada pola, bukan individu
✓ Hapus semua data yang dikumpulkan selama test
```

---

## Deliverable Personnel Pentest

- Tingkat keberhasilan per vektor (% karyawan yang terpancing)
- Departemen/role yang paling rentan
- Waktu rata-rata karyawan melapor ke IT/security
- Rekomendasi training dan awareness
- Simulasi training untuk karyawan yang perlu improvement
