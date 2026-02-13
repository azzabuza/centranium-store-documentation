## **1. Deskripsi Umum**
File ini merupakan **handler API backend** yang berfungsi untuk **membuat transaksi pembayaran** melalui **Midtrans Core API**. File ini dirancang untuk dijalankan di lingkungan **Node.js** dan menangani permintaan HTTP POST dari frontend untuk memproses berbagai metode pembayaran.

---

## **2. Dependensi**

| Nama Modul | Fungsi |
|------------|--------|
| `node-fetch` | Digunakan untuk mengirim HTTP request ke API eksternal (Midtrans) |

---

## **3. Konfigurasi Lingkungan**

### **3.1 Midtrans Configuration**
```javascript
const SERVER_KEY = 'SB-Mid-server-wRZLYZBrMurruPXpxHTVzZhx';
const API_URL = 'https://api.sandbox.midtrans.com/v2/charge';
```
- **SERVER_KEY**: Kredensial server key Midtrans (mode sandbox)
- **API_URL**: Endpoint untuk melakukan charge transaksi ke Midtrans

### **3.2 Domain Configuration**
```javascript
const MY_DOMAIN = process.env.NEXT_PUBLIC_BASE_URL || 'https://centranium.com';
```
- Mengambil domain dari environment variable atau menggunakan default `centranium.com`
- Digunakan untuk **callback URL** dan **notification URL**

---

## **4. Fungsi Utama**
```javascript
module.exports = async (req, res) => { ... }
```
**Handler utama** yang diekspor sebagai modul. Menerima request dan mengembalikan response.

---

## **5. Fitur-Fitur dan Penjelasan Kode**

### **5.1 CORS Handler**
```javascript
res.setHeader('Access-Control-Allow-Origin', '*');
res.setHeader('Access-Control-Allow-Methods', 'POST, OPTIONS');
res.setHeader('Access-Control-Allow-Headers', 'Content-Type');
```
**Fitur:** Mengizinkan akses lintas domain.  
**Penjelasan:** Mengatur header CORS agar frontend dari domain mana pun dapat mengakses endpoint ini.

---

### **5.2 Method Handler**
```javascript
if (req.method === 'OPTIONS') return res.status(200).end();
if (req.method !== 'POST') return res.status(405).json({ error: 'Method not allowed' });
```
**Fitur:** Validasi metode HTTP.  
**Penjelasan:**  
- `OPTIONS`: Untuk preflight request CORS  
- `POST`: Satu-satunya metode yang diizinkan  
- Selain POST: Mengembalikan status `405 Method Not Allowed`

---

### **5.3 Validasi Data Input**
```javascript
try {
    transactionData = req.body;
} catch (e) {
    return res.status(400).json({ success: false, error: 'Invalid Data' });
}
```
**Fitur:** Memastikan data yang dikirim valid JSON.  
**Penjelasan:** Jika parsing body gagal, akan mengembalikan error `400 Bad Request`.

---

### **5.4 Notification URL Builder**
```javascript
const notificationUrl = `${MY_DOMAIN}/api/midtrans-notification`;
```
**Fitur:** Membangun endpoint notifikasi untuk Midtrans.  
**Penjelasan:** URL ini akan dipanggil Midtrans saat status pembayaran berubah.

---

### **5.5 Debug Logging**
```javascript
console.log(`[DEBUG TRANSACTION] Order ${transactionData.transaction_details.order_id} sent URL: ${notificationUrl}`);
```
**Fitur:** Logging untuk debugging.  
**Penjelasan:** Mencatat order ID dan URL notifikasi yang dikirim ke Midtrans.

---

### **5.6 Validasi dan Konversi Amount**
```javascript
if(transactionData.transaction_details && transactionData.transaction_details.gross_amount) {
    transactionData.transaction_details.gross_amount = parseInt(transactionData.transaction_details.gross_amount);
}
```
**Fitur:** Memastikan format nominal benar.  
**Penjelasan:** Midtrans membutuhkan nominal dalam tipe **integer**, bukan string.

---

### **5.7 Pembuatan Payload Midtrans**
```javascript
let coreApiPayload = {
    payment_type: '',
    transaction_details: transactionData.transaction_details,
    item_details: transactionData.item_details,
    customer_details: transactionData.customer_details,
    custom_field1: transactionData.user_id,
    callbacks: {
        notification_url: notificationUrl
    }
};
```
**Fitur:** Membangun struktur data untuk dikirim ke Midtrans.  
**Penjelasan Komponen:**
- `payment_type`: Akan diisi berdasarkan metode pembayaran
- `transaction_details`: Berisi order_id dan gross_amount
- `item_details`: Detail produk yang dibeli
- `customer_details`: Informasi pelanggan
- `custom_field1`: Menyimpan user_id untuk pelacakan
- `callbacks.notification_url`: URL notifikasi

---

### **5.8 Payment Method Mapping**

#### **Bank Transfer (BCA, BNI, BRI, CIMB, BSI, Danamon, Seabank)**
```javascript
case 'bca_va':
case 'bni_va':
    ...
    coreApiPayload.payment_type = 'bank_transfer';
    coreApiPayload.bank_transfer = { bank: pm.replace('_va', '') };
```
**Fitur:** Menangani pembayaran Virtual Account.  
**Penjelasan:** Menghapus suffix `_va` untuk mendapatkan nama bank.

---

#### **Permata Bank**
```javascript
case 'permata_va':
    coreApiPayload.payment_type = 'bank_transfer';
    coreApiPayload.bank_transfer = { bank: 'permata' };
```
**Fitur:** Khusus untuk Permata Virtual Account.

---

#### **Mandiri Virtual Account (E-Channel)**
```javascript
case 'mandiri_va':
    coreApiPayload.payment_type = 'echannel';
    coreApiPayload.echannel = {
        bill_info1: 'Pembayaran',
        bill_info2: 'Centraphone Store'
    };
```
**Fitur:** Menggunakan tipe `echannel` untuk Mandiri VA.  
**Penjelasan:** Mandiri VA memiliki format khusus di Midtrans.

---

#### **QRIS dan Gopay**
```javascript
case 'qris':
case 'gopay':
    coreApiPayload.payment_type = 'gopay';
    coreApiPayload.gopay = {
        enable_callback: true,
        callback_url: `${MY_DOMAIN}/order/history.html`
    };
```
**Fitur:** Pembayaran via QRIS atau Gopay.  
**Catatan:** Saat ini keduanya dipetakan ke `gopay` (perlu dikonfirmasi apakah QRIS sudah terpisah).

---

#### **Metode Tidak Didukung**
```javascript
default:
    return res.status(400).json({ success: false, error: 'Unsupported payment method' });
```
**Fitur:** Validasi metode pembayaran.

---

### **5.9 Autentikasi Midtrans**
```javascript
const authString = Buffer.from(SERVER_KEY + ':').toString('base64');
```
**Fitur:** Membuat header autentikasi.  
**Penjelasan:** Midtrans menggunakan **Basic Auth** dengan format `server_key:` (dengan titik dua di akhir).

---

### **5.10 Request ke Midtrans**
```javascript
const midtransResponse = await fetch(API_URL, {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'Authorization': `Basic ${authString}`,
    },
    body: JSON.stringify(coreApiPayload),
});
```
**Fitur:** Mengirim permintaan charge ke Midtrans.  
**Penjelasan:** Menggunakan `fetch` untuk mengirim data ke endpoint Midtrans.

---

### **5.11 Penanganan Respons Midtrans**

#### **Sukses**
```javascript
return res.status(201).json({
    success: true,
    order_id: transactionData.transaction_details.order_id,
    payment_result: responseData
});
```
**Fitur:** Mengembalikan data pembayaran ke frontend.  
**Status:** `201 Created`

---

#### **Gagal dari Midtrans**
```javascript
console.error("[MIDTRANS ERROR]", responseData);
return res.status(midtransResponse.status).json({
    success: false,
    error: responseData.status_message || 'Midtrans Error',
    details: responseData
});
```
**Fitur:** Menangani error dari pihak Midtrans.

---

#### **Server Error**
```javascript
catch (error) {
    console.error('Server Error:', error);
    return res.status(500).json({ success: false, error: 'Failed to connect payment gateway' });
}
```
**Fitur:** Menangani error internal server.

---

## **6. Alur Kerja Program**

```
[FRONTEND] 
    ↓ (POST Request)
[VALIDASI INPUT]
    ↓
[BUILD NOTIFICATION URL]
    ↓
[VALIDASI & KONVERSI AMOUNT]
    ↓
[BUILD PAYLOAD MIDTRANS]
    ↓
[MAPPING METODE PEMBAYARAN]
    ↓
[KIRIM KE MIDTRANS API]
    ↓
[PROSES RESPONS]
    ├── Sukses → Kembalikan 201 + Data Pembayaran
    └── Gagal  → Kembalikan Error sesuai kasus
```

---

## **7. Metode Pembayaran yang Didukung**

| Kode | Metode | Tipe Midtrans |
|------|--------|---------------|
| `bca_va` | BCA Virtual Account | bank_transfer |
| `bni_va` | BNI Virtual Account | bank_transfer |
| `bri_va` | BRI Virtual Account | bank_transfer |
| `cimb_va` | CIMB Virtual Account | bank_transfer |
| `bsi_va` | BSI Virtual Account | bank_transfer |
| `danamon_va` | Danamon Virtual Account | bank_transfer |
| `seabank_va` | Seabank Virtual Account | bank_transfer |
| `permata_va` | Permata Virtual Account | bank_transfer |
| `mandiri_va` | Mandiri Virtual Account | echannel |
| `qris` | QRIS | gopay* |
| `gopay` | GoPay | gopay |

*\*Perlu penyesuaian jika QRIS sudah memiliki tipe sendiri*

---

## **8. Environment Variables**

| Variabel | Keterangan | Default |
|----------|------------|---------|
| `NEXT_PUBLIC_BASE_URL` | Domain aplikasi | `https://centranium.com` |

---

## **9. Potensi Pengembangan**

1. **Pemisahan QRIS dan Gopay** — Saat ini masih 1 tipe (`gopay`)
2. **Penambahan metode lain** — Contoh: Credit Card, Indomaret, Alfamart
3. **Validasi lebih ketat** — Memastikan semua field wajib terisi
4. **Rate limiting** — Mencegah spam request

---

Dokumentasi ini disusun berdasarkan komentar yang ada dalam kode serta hasil analisis struktur dan alur program.
