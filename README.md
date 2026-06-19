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
| [scenarios/16-daftar-tools.md](scenarios/16-daftar-tools.md) | Daftar lengkap tools, instalasi, dan matriks tools per skenario |

---

## Prasyarat

- VSCode dengan Claude Code terinstal
- Python 3.8+, tools: nmap, gobuster, nikto, sqlmap, wpscan
- Target yang **diotorisasi** atau platform lab (TryHackMe, HackTheBox)

---

> **Legal Notice:** Semua teknik hanya untuk digunakan pada sistem yang Anda miliki atau memiliki izin eksplisit tertulis. Penggunaan tanpa izin adalah ilegal dan melanggar UU ITE Indonesia.
