# Skenario 9: Aplikasi Internet Publik — White Box

**Akses:** Internet publik + akses penuh ke repository dan infrastruktur  
**Metode:** White Box — source code, CI/CD, cloud config, semua akun  
**Contoh target:** `https://www.targetapp.co.id` + GitHub repo + AWS console access

---

## Informasi Awal yang Tersedia

| Item | Status |
|------|--------|
| Source code (GitHub/GitLab) | ✓ Full repo + history |
| Cloud console (AWS/GCP/Azure) | ✓ Read access |
| Semua akun (admin, DB, API) | ✓ |
| CI/CD pipeline config | ✓ |
| Infrastruktur (Terraform/Ansible) | ✓ |
| Third-party API keys | ✓ |
| Architecture diagram | ✓ |

---

## Fokus Khusus White Box Internet

Aplikasi internet produksi menambahkan dimensi:
1. **Cloud security** — misconfigured S3, IAM over-permission, exposed secrets
2. **CI/CD security** — credential injection, pipeline hijacking
3. **Supply chain** — dependency dengan CVE, malicious package
4. **Secret scanning** — API keys tercampur di git history
5. **Third-party integrations** — OAuth, payment gateway, email service

---

## Phase 1: Git Repository Secret Scanning

```bash
# Clone repository
git clone https://github.com/org/targetapp .

# Gitleaks — scan seluruh git history
gitleaks detect --source . --report-format json \
  --report-path vulnerabilities/gitleaks.json -v

# TruffleHog — scan commits
trufflehog git file://. --json \
  > vulnerabilities/trufflehog.json

# Manual scan git history
git log --all --oneline | head -100
git log -p --all -- "*.env" | grep "^+"
git log -p --all | grep -E "password|secret|key|token|api_key" | head -50

# Cek branch dan tag tersembunyi
git branch -a
git tag -l
git stash list
```

**Prompt Claude Code:**
```
Gitleaks dan TruffleHog menemukan beberapa potential secrets di git history
dari repo https://github.com/org/targetapp:

[PASTE OUTPUT GITLEAKS/TRUFFLEHOG]

Untuk setiap temuan:
1. Apakah ini valid credential atau false positive?
2. Jika valid, apa dampak jika credential ini masih aktif?
3. Cara verifikasi apakah credential masih aktif tanpa menggunakannya secara merusak
4. Prioritas invalidasi/rotasi
5. Rekomendasi prevent credential leakage di masa depan
```

---

## Phase 2: Cloud Security Audit (AWS)

```bash
# AWS CLI audit (dengan read-only credentials yang diberikan)

# 1. Cek S3 buckets — apakah ada yang public?
aws s3 ls
aws s3api list-buckets --query 'Buckets[].Name' --output text
for bucket in $(aws s3api list-buckets --query 'Buckets[].Name' --output text); do
    acl=$(aws s3api get-bucket-acl --bucket $bucket 2>/dev/null)
    public=$(echo $acl | python3 -c "import sys,json; data=json.load(sys.stdin); \
      print('PUBLIC' if any(g.get('URI','').endswith('AllUsers') \
      for g in [grant['Grantee'] for grant in data['Grants']]) else 'private')" 2>/dev/null)
    echo "$bucket: $public"
done

# 2. Cek IAM policies — apakah ada over-permissive?
aws iam list-users --output table
aws iam get-account-authorization-details \
  --filter User --output json > vulnerabilities/iam_audit.json

# 3. Cek Security Groups — port terbuka ke 0.0.0.0/0?
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]]' \
  --output table

# 4. Cek CloudTrail — logging aktif?
aws cloudtrail describe-trails
aws cloudtrail get-trail-status --name main-trail

# 5. RDS — apakah publicly accessible?
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,PubliclyAccessible,StorageEncrypted]' \
  --output table
```

**Prompt Claude Code:**
```
AWS audit dari white box pentest targetapp.co.id:

S3 buckets:
- targetapp-production: private
- targetapp-backups: PUBLIC ← masalah!
- targetapp-assets: public (CDN)

IAM: App user memiliki policy AdministratorAccess ← terlalu luas!

Security Groups:
- sg-web: port 22 (SSH) open ke 0.0.0.0/0 ← berbahaya!
- sg-db: port 3306 open ke 0.0.0.0/0 ← sangat berbahaya!

RDS: PubliclyAccessible = True, StorageEncrypted = False

Analisis dampak dan rekomendasi untuk setiap temuan.
Berikan severity dan AWS-specific remediation steps.
```

---

## Phase 3: CI/CD Pipeline Security

```yaml
# Review GitHub Actions / GitLab CI config

# .github/workflows/deploy.yml — cari masalah:
# - Secrets hardcoded
# - Insecure checkout (arbitrary code execution)
# - Over-permissive tokens
# - Unverified actions (supply chain)
```

**Prompt Claude Code:**
```
Review GitHub Actions CI/CD pipeline dari targetapp.co.id:

[PASTE .github/workflows/deploy.yml]

Identifikasi security issues:
1. Apakah ada secrets hardcoded di workflow?
2. Apakah menggunakan actions dari third party tanpa pin commit SHA?
   (rawan supply chain attack jika pakai @main atau @v1)
3. Apakah GITHUB_TOKEN memiliki permission yang terlalu luas?
4. Apakah ada command injection vulnerability dari input external?
5. Apakah pull_request_target digunakan? (berbahaya untuk fork PRs)
6. Rekomendasi perbaikan untuk setiap temuan
```

---

## Phase 4: Infrastructure as Code Review

```bash
# Scan Terraform configs
checkov -d infrastructure/terraform/ \
  --output json > vulnerabilities/checkov_terraform.json

# Scan Docker/Kubernetes configs
checkov -d kubernetes/ --framework kubernetes \
  --output json > vulnerabilities/checkov_k8s.json

# Hadolint untuk Dockerfile
hadolint Dockerfile

# Trivy untuk container image scan
trivy image targetapp/backend:latest \
  --format json --output vulnerabilities/trivy.json
```

**Prompt Claude Code:**
```
Checkov scan dari Terraform configs:
[PASTE CHECKOV OUTPUT]

Dockerfile issues dari Hadolint:
[PASTE HADOLINT OUTPUT]

Trivy container scan summary:
Critical: 3, High: 8, Medium: 15

Dari temuan ini:
1. Mana yang paling mudah dieksploitasi dari internet?
2. Apakah ada misconfiguration yang bisa menyebabkan privilege escalation ke host?
3. Container image dengan CVE critical — exploit yang tersedia?
4. Rekomendasi IaC security best practices
```

---

## Phase 5: Third-Party Integration Testing

```bash
# Test API keys yang ditemukan
# Stripe API key
curl -s "https://api.stripe.com/v1/customers" \
  -H "Authorization: Bearer sk_live_XXXX" | python3 -m json.tool | head -20

# Twilio SMS API
curl -X GET "https://api.twilio.com/2010-04-01/Accounts/ACXXX/Messages.json" \
  -u "ACXXX:auth_token" | python3 -m json.tool | head -20

# SendGrid / Email API
curl -s --request GET \
  --url https://api.sendgrid.com/v3/suppression/bounces \
  --header "Authorization: Bearer SG.XXXX"
```

**Prompt Claude Code:**
```
White box pentest menemukan API keys berikut di konfigurasi/git history
dari targetapp.co.id (authorized pentest):

Stripe: sk_live_XXXXXXXXXXXX
Twilio: auth_token dari .env
SendGrid: SG.XXXXXXXXXX

Untuk setiap API key:
1. Cara verify apakah masih aktif (tanpa melakukan action destruktif)
2. Apa yang bisa dilakukan penyerang jika key ini bocor?
   (read customer data, send SMS/email ke semua user, charge kartu?)
3. Severity dari perspektif bisnis
4. Cara segera mitigasi
```

---

## Phase 6: Complete White Box Laporan Internet

**Prompt Claude Code:**
```
Generate laporan white box pentest komprehensif untuk targetapp.co.id.
Klien: perusahaan e-commerce dengan ~100.000 users.

Temuan:
CRITICAL:
1. AWS S3 bucket 'targetapp-backups' public → berisi DB dump 6 bulan lalu
2. API key Stripe sk_live bocor di git commit 2 tahun lalu
3. MySQL RDS publicly accessible tanpa enkripsi

HIGH:
4. SSH port 22 open ke seluruh internet
5. IAM user dengan AdministratorAccess
6. JWT secret lemah (12345678) → bisa forge token admin

MEDIUM:
7. CI/CD pipeline menggunakan actions tidak di-pin (supply chain risk)
8. 3 library dengan CVE High (remote code execution)
9. Container image dengan kernel exploit vulnerability

LOW:
10. Security headers tidak lengkap (CSP, HSTS)
11. Informasi versi di HTTP headers

Buat laporan executive + teknis dengan:
- Business impact setiap temuan
- Estimasi CVSS score
- Remediation timeline (immediate/1 minggu/1 bulan)
- Referensi standar (OWASP, CIS Benchmark, AWS Well-Architected)
```

---

## Checklist White Box Internet

```
[ ] Git history secret scanning (gitleaks, trufflehog)
[ ] Dependency vulnerability audit (composer, npm, pip)
[ ] Static code analysis (semgrep, CodeQL)
[ ] Cloud security audit (AWS/GCP/Azure)
[ ] S3/Storage bucket permission review
[ ] IAM over-permission analysis
[ ] Security group / firewall rules review
[ ] CI/CD pipeline security review
[ ] IaC scan (Terraform, Kubernetes manifests)
[ ] Container image vulnerability scan (Trivy)
[ ] Third-party API key validation
[ ] Database encryption & access control
[ ] Logging & monitoring capability review
[ ] Dynamic testing berdasarkan code review
[ ] Laporan dengan business impact + remediation timeline
[ ] Briefing executive presentation
```
