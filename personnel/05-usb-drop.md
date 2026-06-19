# Personnel Pentest: USB Drop Attack

**Tujuan:** Test apakah karyawan akan mencolokkan USB yang "ditemukan"  
**Durasi:** 1–3 hari  
**Tools:** USB Rubber Ducky, Bash Bunny, atau USB biasa dengan autorun script

---

## Mengapa USB Drop Efektif

- Manusia penasaran → 45-60% USB yang dijatuhkan akan dicolokan (riset Google/Elie Bursztein)
- "Finders keepers" mentality
- Mungkin ada data menarik di dalamnya
- Di area parkiran/lobby → penyerang tidak perlu masuk gedung

---

## Phase 1: Persiapan USB

```bash
# Tipe USB yang digunakan untuk test (authorized pentest):

# Opsi A: USB biasa dengan file "menarik"
# - Isi dengan file bernama: "Gaji_Karyawan_2024.xlsx", "Daftar_Bonus.pdf"
# - File sebenarnya berisi tracking pixel atau link ke server
# - Saat dibuka → server mencatat siapa yang buka

# Opsi B: USB dengan HID payload (Rubber Ducky)
# - Begitu dicolok: eksekusi script
# - Kirim hostname, username, IP ke server tracking
# - Tidak perlu karyawan klik apapun

# Opsi C: File dengan macro tracking
# Buat Word/Excel dengan macro yang ping server saat dibuka
# (untuk test kesadaran, bukan exploit nyata)

# Label USB yang menarik (untuk meningkatkan chance dicolokan):
# "RAHASIA - GAJI 2024"
# "Foto Liburan Kantor"
# "Rencana Strategis 2025"
# "BONUS Q4"
```

**Prompt Claude Code:**
```
USB drop test di kantor targetcorp.co.id (authorized).
Tujuan: hanya tracking (tidak ada payload berbahaya, hanya laporan ke server).

Buatkan:
1. Script PowerShell ringan yang kirim hostname + username + timestamp ke webhook saya
   (tanpa trigger antivirus, tanpa modifikasi sistem)
2. Cara embed script ke shortcut (.lnk file) yang terlihat seperti folder
3. Cara buat file Word dengan macro tracking (kirim beacon saat dibuka)
4. Label/nama USB yang paling mungkin bikin karyawan penasaran
5. Lokasi drop yang strategis di kantor (lobby, parkiran, lift, toilet)
```

---

## Phase 2: Penempatan USB

```
Strategi penempatan USB:

Lokasi efektif:
- Parkiran karyawan (sering ditemukan seolah jatuh dari tas)
- Lift → taruh di sudut lantai lift
- Lobby sebelum pintu masuk
- Kantin / area makan
- Toilet (di atas wastafel atau dekat cermin)
- Di atas ATM gedung

Teknik penempatan:
- USB ditaruh bukan di tempat yang "terlalu" mencolok
- Tidak di tengah meja — lebih ke sudut atau dekat dinding
- Bisa ditaruh beberapa sekaligus di lokasi berbeda

Dokumentasikan:
- Waktu penempatan
- Lokasi tepat (foto)
- Waktu pertama kali USB diakses (dari server log)
- USB mana yang dicolokan (jika dibedakan per lokasi)
```

---

## Phase 3: Tracking & Monitoring

```bash
# Setup tracking server sederhana
# Python webhook untuk catat beacon

from flask import Flask, request
import json, datetime

app = Flask(__name__)

@app.route('/beacon', methods=['GET', 'POST'])
def beacon():
    data = {
        "timestamp": datetime.datetime.now().isoformat(),
        "ip": request.remote_addr,
        "user_agent": request.headers.get('User-Agent'),
        "data": request.args.to_dict() or request.json
    }
    print(f"[!] USB accessed: {json.dumps(data, indent=2)}")
    with open('usb_log.json', 'a') as f:
        f.write(json.dumps(data) + '\n')
    return "OK", 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)

# Monitor real-time
tail -f usb_log.json
```

---

## Phase 4: Analisis Hasil

**Prompt Claude Code:**
```
Hasil USB drop test (10 USB, 3 hari):

USB yang dicolokan: 6/10 (60%)
USB yang hanya dibuka file-nya: 2/10 (20%)
USB yang dilaporkan ke IT: 1/10 (10%)
USB tidak ditemukan/tidak dicolokan: 1/10 (10%)

Log server:
- USB #1 (parkiran): dicolokan 47 menit setelah ditaruh
  Hostname: FINANCE-PC-007, User: budi.santoso
- USB #3 (lobby): dicolokan 2 jam setelah ditaruh
  Hostname: RECEPT-PC-001, User: reception
- USB #5 (kantin): file dibuka, tapi tidak execute (antivirus blokir macro)
  User: john.doe — AV masih bekerja!

Buat analisis:
1. 60% USB dicolokan — benchmark industry? (Google study: 48%)
2. Departemen/role mana yang paling berisiko?
3. Antivirus di USB #5 berhasil blokir — apakah proteksi cukup?
4. Rekomendasi: USB port blocking policy
5. Training yang diperlukan: "jangan colokan USB yang tidak dikenal"
```

---

## Checklist USB Drop Pentest

```
[ ] Siapkan USB dengan payload tracking (bukan destruktif)
[ ] Label USB dengan nama yang "menarik"
[ ] Setup tracking server
[ ] Drop USB di beberapa lokasi strategis
[ ] Pantau server selama 48-72 jam
[ ] Catat: lokasi, waktu dicolokan, siapa yang colokan
[ ] Collect USB kembali setelah test selesai
[ ] Hapus semua payload dari USB
[ ] Reveal ke management
[ ] Laporan: persentase, lokasi paling berbahaya, rekomendasi
```

---

## Rekomendasi untuk Laporan

```
Teknis:
- Blokir USB storage via Group Policy (tapi izinkan keyboard/mouse)
- Endpoint DLP untuk monitor USB usage
- Antivirus dengan real-time protection dan USB scanning

Non-teknis (awareness):
- Training: "jangan colokan USB yang tidak dikenal, bahkan di kantor"
- Poster awareness di area kantor
- Prosedur: USB yang ditemukan → serahkan ke IT Security
- Simulasi rutin setiap 6 bulan
```

---

## Durasi Estimasi

| Fase | Durasi |
|------|-----------|----------|
| Persiapan USB + server | 0.5–1 hari | 1–2 hari |
| Drop + monitoring (48-72 jam) | 2–3 hari | 2–3 hari (sama) |
| Laporan | 0.5 hari | 1 hari |
| **Total** | **3–5 hari** | **4–6 hari** |
