# Workflow CTF (Capture The Flag) dengan Claude Code

> Panduan penggunaan Claude Code untuk menyelesaikan tantangan CTF secara sistematis.

---

## Setup CTF Environment di VSCode

### Struktur Folder CTF

```
ctf-[nama-event]/
├── challenges/
│   ├── web/
│   ├── pwn/
│   ├── crypto/
│   ├── forensics/
│   ├── misc/
│   └── reversing/
├── tools/
├── notes.md
└── flags.txt
```

**Buat dengan satu perintah ke Claude Code:**
```
Buatkan struktur folder untuk CTF event bernama "HackFest2024" dengan kategori:
web, pwn, crypto, forensics, misc, reversing
```

---

## Pendekatan per Kategori

### Web Challenges

**Initial Reconnaissance:**
```bash
# Lihat source HTML
curl -s [URL] | grep -iE 'password|flag|secret|token|key|admin'

# Check robots.txt
curl -s [URL]/robots.txt

# Check HTTP headers
curl -I [URL]

# Cari komentar HTML
curl -s [URL] | grep '<!--'
```

**Prompt ke Claude Code:**
```
Web CTF challenge di [URL]. Source code yang saya temukan:
[PASTE SOURCE]

Header HTTP:
[PASTE HEADERS]

Identifikasi vulnerability dan berikan exploitation steps.
```

---

### Cryptography Challenges

**Identifikasi Cipher:**

```python
# crypto/identify_cipher.py
# Minta Claude Code generate ini

import re
import base64
from collections import Counter

def identify_cipher(ciphertext):
    hints = []
    
    # Check Base64
    if re.match(r'^[A-Za-z0-9+/]+=*$', ciphertext.strip()):
        hints.append("Kemungkinan Base64")
        try:
            decoded = base64.b64decode(ciphertext)
            hints.append(f"Base64 decoded: {decoded[:50]}")
        except:
            pass
    
    # Check hex
    if re.match(r'^[0-9A-Fa-f]+$', ciphertext.strip()):
        hints.append("Kemungkinan Hex encoding")
    
    # Check untuk angka (mungkin ASCII codes)
    numbers = re.findall(r'\d+', ciphertext)
    if numbers and all(0 <= int(n) <= 127 for n in numbers):
        hints.append(f"Angka-angka dalam range ASCII: {[chr(int(n)) for n in numbers[:10]]}")
    
    # Frequency analysis untuk substitution cipher
    letters_only = re.sub(r'[^A-Za-z]', '', ciphertext.upper())
    if letters_only:
        freq = Counter(letters_only)
        most_common = freq.most_common(5)
        hints.append(f"Karakter paling sering: {most_common}")
        hints.append("Jika teks Inggris: E,T,A,O,I adalah huruf paling umum")
    
    return '\n'.join(hints)

ciphertext = input("Masukkan ciphertext: ")
print(identify_cipher(ciphertext))
```

**Prompt analisis crypto:**
```
CTF Crypto challenge. Ciphertext: [CIPHERTEXT]
Konteks dari soal: [DESKRIPSI]

Analisis:
1. Jenis cipher apa ini?
2. Bagaimana cara mendekripsinya?
3. Berikan Python code untuk decrypt
```

---

### Forensics Challenges

**Checklist Forensics dengan Claude Code:**

```bash
#!/bin/bash
# forensics/initial_analysis.sh

FILE=$1
echo "=== File Analysis: $FILE ==="

echo -e "\n[1] File Type:"
file "$FILE"

echo -e "\n[2] Strings (interesting):"
strings "$FILE" | grep -iE 'flag|ctf|password|key|secret|http|ftp'

echo -e "\n[3] Metadata (exiftool):"
exiftool "$FILE" 2>/dev/null

echo -e "\n[4] Hex dump (first 256 bytes):"
xxd "$FILE" | head -16

echo -e "\n[5] Binwalk analysis:"
binwalk "$FILE"

echo -e "\n[6] Entropy analysis:"
binwalk -E "$FILE" 2>/dev/null
```

**Prompt untuk file mencurigakan:**
```
Forensics CTF challenge. Output analisis file:

file command: [OUTPUT]
strings output: [INTERESTING STRINGS]
exiftool: [METADATA]
binwalk: [BINWALK OUTPUT]

File adalah [gambar/PDF/ZIP/binary]. Apa yang harus saya cari selanjutnya?
```

---

### Binary Exploitation (PWN)

**Initial Analysis:**

```python
# pwn/initial_pwn.py
import subprocess

def analyze_binary(binary_path):
    print(f"[*] Analyzing: {binary_path}")
    
    # File info
    result = subprocess.run(['file', binary_path], capture_output=True, text=True)
    print(f"\n[File] {result.stdout}")
    
    # Security features
    result = subprocess.run(['checksec', '--file=' + binary_path], 
                          capture_output=True, text=True)
    print(f"[Security]\n{result.stdout}")
    
    # Interesting strings
    result = subprocess.run(['strings', binary_path], capture_output=True, text=True)
    interesting = [s for s in result.stdout.split('\n') 
                   if any(kw in s.lower() for kw in ['flag', 'win', 'shell', 'system', '/bin'])]
    print(f"\n[Interesting strings]\n" + '\n'.join(interesting[:20]))

analyze_binary('./challenge_binary')
```

**Prompt PWN:**
```
Binary PWN challenge. Informasi:

checksec output: [OUTPUT]
file output: [OUTPUT]
Fungsi yang menarik (dari Ghidra/radare2): [DECOMPILED CODE]

Vulnerability apa yang ada? Bagaimana exploit skeleton-nya?
Gunakan pwntools.
```

---

## Claude Code untuk Reverse Engineering

### Template Analisis Ghidra Output

**Prompt:**
```
Ini adalah decompiled C code dari Ghidra untuk binary CTF challenge:

[PASTE DECOMPILED CODE]

1. Apa yang dilakukan program ini?
2. Di mana logika validasi flag/password-nya?
3. Bagaimana cara mendapatkan flag tanpa menjalankan program?
4. Apakah ada static key atau hardcoded comparison?
```

### Keying Otomatis dengan Z3

**Minta Claude Code generate solver:**
```
Ini adalah kondisi validasi dari reverse engineering challenge:

[KONDISI/CONSTRAINTS]

Buatkan Python script menggunakan Z3 solver untuk menemukan input
yang memenuhi semua constraints ini.
```

---

## Notes & Documentation Workflow

### Template Notes per Challenge

```markdown
# Challenge: [NAMA]
**Kategori:** Web/Crypto/Forensics/etc  
**Points:** [POIN]  
**Status:** In Progress / Solved

## Deskripsi
[Copy deskripsi challenge]

## Initial Observations
- 
- 

## Attempts
### Attempt 1: [Pendekatan]
- Result: 
- Command: `[command yang digunakan]`

## Solution
[Langkah solution jika sudah solved]

## Flag
`flag{...}`

## Lessons Learned
- 
```

**Minta Claude Code generate ini:**
```
Buatkan bash script yang otomatis membuat file notes.md 
untuk setiap challenge CTF baru dengan template standard
yang mencakup: nama, kategori, poin, deskripsi, sections untuk attempts dan solution.
```

---

## Tips CTF Cepat dengan Claude Code

1. **Screenshot → Analisis**: Paste screenshot soal CTF ke Claude Code, minta analisis langsung
2. **"Apa yang dicoba selanjutnya?"**: Setelah dead end, ceritakan apa yang sudah dicoba
3. **Binary to Text**: Paste binary/hex/base64, minta decode chain
4. **Pattern Recognition**: "Apakah ini terlihat seperti format encoding tertentu?"
5. **Script Runner**: Minta Claude Code buat script, jalankan di terminal VSCode, paste hasilnya balik

---

## Resource CTF

| Platform | URL | Keterangan |
|----------|-----|------------|
| TryHackMe | tryhackme.com | Beginner friendly, guided |
| HackTheBox | hackthebox.com | Lebih challenging |
| PicoCTF | picoctf.org | CTF untuk pelajar |
| CTFtime | ctftime.org | Kalender CTF event |
| OverTheWire | overthewire.org | Wargames/CTF |
