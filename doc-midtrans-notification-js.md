---

# **Dokumentasi Kode: midtrans-notification.js**

## **1. Deskripsi Umum**
File ini merupakan **handler webhook/notifikasi** yang berfungsi untuk **menerima dan memproses notifikasi status pembayaran** dari Midtrans. File ini bertugas memperbarui status pesanan di **Firebase Realtime Database** berdasarkan notifikasi yang dikirim oleh Midtrans.

---

## **2. Dependensi**

| Nama Modul | Fungsi |
|------------|--------|
| `firebase-admin` | Mengelola koneksi dan operasi ke Firebase Realtime Database |

---

## **3. Konfigurasi Lingkungan**

### **3.1 Midtrans Server Key**
```javascript
const MIDTRANS_SERVER_KEY = 'SB-Mid-server-wRZLYZBrMurruPXpxHTVzZhx';
```
**Fungsi:** Menyimpan server key Midtrans untuk mode sandbox.  
**Catatan:** Saat ini tidak digunakan langsung dalam kode, namun tersedia untuk keperluan verifikasi signature (dapat dikembangkan lebih lanjut).

---

### **3.2 Firebase Configuration**
```javascript
databaseURL: "https://centranium-store-default-rtdb.asia-southeast1.firebasedatabase.app"
```
**Fungsi:** URL Firebase Realtime Database yang digunakan.

---

## **4. Inisialisasi Firebase Admin SDK**

### **4.1 Inisialisasi Bersyarat**
```javascript
if (!admin.apps.length) {
    // Inisialisasi hanya jika belum ada instance Firebase
}
```
**Fitur:** Mencegah inisialisasi ganda.  
**Penjelasan:** Memeriksa apakah Firebase Admin SDK sudah diinisialisasi sebelumnya.

---

### **4.2 Pengambilan Kredensial dari Environment**
```javascript
const envJson = process.env.FIREBASE_ADMIN_JSON;
if (!envJson) throw new Error("FIREBASE_ADMIN_JSON is empty");

const serviceAccount = JSON.parse(envJson);
```
**Fitur:** Mengambil konfigurasi Firebase dari environment variable.  
**Penjelasan:** File kredensial Firebase tidak disimpan di kode, melainkan di **environment variable** `FIREBASE_ADMIN_JSON` untuk keamanan.

---

### **4.3 Normalisasi Private Key**
```javascript
if (serviceAccount.private_key) {
    serviceAccount.private_key = serviceAccount.private_key.replace(/\\n/g, '\n');
}
```
**Fitur:** Memperbaiki format private key.  
**Penjelasan:** Private key sering mengalami escape karakter (`\\n`) saat disimpan sebagai JSON string. Kode ini mengembalikannya ke format baris baru yang benar (`\n`).

---

### **4.4 Inisialisasi Aplikasi**
```javascript
admin.initializeApp({
    credential: admin.credential.cert(serviceAccount),
    databaseURL: "https://centranium-store-default-rtdb.asia-southeast1.firebasedatabase.app"
});
```
**Fitur:** Memulai koneksi ke Firebase.  
**Penjelasan:** Menggunakan metode sertifikat untuk autentikasi ke Firebase services.

---

### **4.5 Penanganan Error Inisialisasi**
```javascript
catch (error) {
    console.error("[INIT ERROR] Firebase init failed:", error.message);
}
```
**Fitur:** Menangani kegagalan inisialisasi Firebase.  
**Penjelasan:** Aplikasi tetap berjalan meskipun Firebase gagal diinisialisasi, namun tidak dapat memperbarui database.

---

## **5. Fungsi Utama**
```javascript
module.exports = async (req, res) => { ... }
```
**Handler utama** yang menangani request webhook dari Midtrans.

---

## **6. Fitur-Fitur dan Penjelasan Kode**

### **6.1 Logging Incoming Request**
```javascript
console.log("[INCOMING] Notification received from:", req.headers['user-agent']);
```
**Fitur:** Mencatat notifikasi masuk.  
**Penjelasan:** Membantu debugging dengan mencatat user-agent pengirim (diharapkan dari Midtrans).

---

### **6.2 Validasi Metode HTTP**
```javascript
if (req.method !== 'POST') return res.status(405).send('Method Not Allowed');
```
**Fitur:** Membatasi hanya metode POST.  
**Penjelasan:** Midtrans mengirim notifikasi melalui metode POST.

---

### **6.3 Ekstraksi Data Notifikasi**
```javascript
const notif = req.body;
const orderId = notif.order_id;
const status = notif.transaction_status;
const userId = notif.custom_field1;
```
**Fitur:** Mengambil data penting dari payload Midtrans.  
**Penjelasan Komponen:**
- `order_id`: ID pesanan yang dibuat saat transaksi
- `transaction_status`: Status pembayaran dari Midtrans
- `custom_field1`: Field custom yang berisi `user_id` (dikirim dari `create-core-transaction.js`)

---

### **6.4 Debug Payload**
```javascript
console.log("[DEBUG PAYLOAD]", JSON.stringify({
    order_id: notif.order_id,
    status: notif.transaction_status,
    custom_field1: notif.custom_field1
}));
```
**Fitur:** Mencatat payload notifikasi.  
**Penjelasan:** Memudahkan pelacakan notifikasi yang masuk.

---

### **6.5 Validasi Order ID**
```javascript
if (!orderId) {
    console.error("[ERROR] No Order ID in notification");
    return res.status(400).send('No Order ID');
}
```
**Fitur:** Memastikan notifikasi memiliki order_id.  
**Penjelasan:** Order ID adalah kunci utama untuk mengidentifikasi pesanan.

---

### **6.6 Status Mapping**

#### **Status Mapping Logic**
```javascript
let dbStatus = 'waiting';

if (['capture', 'settlement', 'success'].includes(status)) {
    dbStatus = 'settlement';
} else if (['deny', 'cancel', 'expire', 'failure'].includes(status)) {
    dbStatus = 'failed';
} else if (status === 'pending') {
    dbStatus = 'waiting';
}
```
**Fitur:** Memetakan status Midtrans ke status internal database.  
**Penjelasan Mapping:**

| Status Midtrans | Status Database | Keterangan |
|-----------------|-----------------|------------|
| `capture` | `settlement` | Pembayaran berhasil (CC capture) |
| `settlement` | `settlement` | Pembayaran berhasil |
| `success` | `settlement` | Pembayaran berhasil |
| `deny` | `failed` | Pembayaran ditolak |
| `cancel` | `failed` | Pembayaran dibatalkan |
| `expire` | `failed` | Pembayaran kadaluarsa |
| `failure` | `failed` | Pembayaran gagal |
| `pending` | `waiting` | Menunggu pembayaran |
| *Lainnya* | `waiting` | Default status |

---

### **6.7 Database Update**

#### **Update dengan User ID**
```javascript
if (userId) {
    await admin.database().ref(`orders/${userId}/${orderId}`).update({
        status: dbStatus,
        transactionStatus: status,
        updatedAt: admin.database.ServerValue.TIMESTAMP
    });
    console.log(`[SUCCESS] Updated orders/${userId}/${orderId} to ${dbStatus}`);
}
```
**Fitur:** Memperbarui status pesanan di Firebase.  
**Penjelasan Path Database:**  
`orders/{userId}/{orderId}`

**Field yang Diupdate:**
| Field | Nilai | Keterangan |
|-------|-------|------------|
| `status` | `dbStatus` | Status internal aplikasi |
| `transactionStatus` | `status` | Status asli dari Midtrans |
| `updatedAt` | `TIMESTAMP` | Waktu update (server Firebase) |

---

#### **Handle Missing User ID**
```javascript
else {
    console.error(`[ERROR] User ID is MISSING. Cannot update database.`);
}
```
**Fitur:** Penanganan jika user_id tidak tersedia.  
**Penjelasan:** Tidak dapat memperbarui database tanpa userId, hanya mencatat error.

---

### **6.8 Response Handler**

#### **Response Sukses**
```javascript
return res.status(200).send('OK');
```
**Fitur:** Mengkonfirmasi penerimaan notifikasi ke Midtrans.  
**Penjelasan:** Midtrans membutuhkan response `200 OK` untuk menganggap notifikasi berhasil terkirim.

---

#### **Response Error**
```javascript
return res.status(400).send('No Order ID');
return res.status(405).send('Method Not Allowed');
return res.status(500).send('Internal Error');
```
**Fitur:** Mengirim response error ke Midtrans.  
**Penjelasan:** Midtrans akan mencoba mengirim ulang notifikasi jika menerima response selain `200 OK`.

---

## **7. Alur Kerja Program**

```
[MIDTRANS]
    ↓ (POST Notification)
[VALIDASI METHOD]
    ↓
[PARSING PAYLOAD]
    ├── order_id
    ├── transaction_status
    └── custom_field1 (user_id)
    ↓
[VALIDASI ORDER ID]
    ↓
[STATUS MAPPING]
    ├── settlement ← capture, settlement, success
    ├── failed    ← deny, cancel, expire, failure
    └── waiting   ← pending, others
    ↓
[VALIDASI USER ID]
    ├── Ada    → Update Database
    └── Tidak  → Log Error
    ↓
[RESPONSE]
    ├── 200 OK     → Sukses
    ├── 400/405/500 → Error
```

---

## **8. Struktur Database Firebase**

```
orders/
└── {userId}/
    └── {orderId}/
        ├── status: "settlement" | "failed" | "waiting"
        ├── transactionStatus: "capture" | "settlement" | "pending" | ...
        └── updatedAt: timestamp
```

---

## **9. Environment Variables**

| Variabel | Keterangan | Wajib |
|----------|------------|-------|
| `FIREBASE_ADMIN_JSON` | JSON string dari service account Firebase Admin SDK | ✅ Ya |

**Contoh Format FIREBASE_ADMIN_JSON:**
```json
{
  "type": "service_account",
  "project_id": "centranium-store",
  "private_key_id": "...",
  "private_key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n",
  "client_email": "...",
  "client_id": "...",
  "auth_uri": "...",
  "token_uri": "..."
}
```

---

## **10. Fitur Keamanan**

### **Sudah Diimplementasikan:**
✅ Inisialisasi Firebase yang aman (credential dari environment)  
✅ Normalisasi private key  
✅ Validasi input dasar  

### **Potensi Pengembangan:**
🔐 **Verifikasi Signature Midtrans** — Memverifikasi bahwa notifikasi benar-benar dari Midtrans menggunakan signature key  
🔐 **Rate Limiting** — Mencegah spam notifikasi  
🔐 **IP Whitelist** — Hanya menerima dari IP Midtrans  
🔐 **Idempotency** — Mencegah update ganda untuk notifikasi yang sama  

---

## **11. Logging Events**

| Event | Level | Keterangan |
|-------|-------|------------|
| `[INIT] Firebase Admin initialized successfully` | Info | Firebase berhasil diinisialisasi |
| `[INIT ERROR] Firebase init failed` | Error | Gagal inisialisasi Firebase |
| `[INCOMING] Notification received from` | Info | Notifikasi masuk |
| `[DEBUG PAYLOAD]` | Debug | Detail payload notifikasi |
| `[ERROR] No Order ID in notification` | Error | Order ID tidak ditemukan |
| `[SUCCESS] Updated orders/...` | Info | Database berhasil diupdate |
| `[ERROR] User ID is MISSING` | Error | User ID tidak ada dalam payload |
| `[CRITICAL ERROR]` | Error | Error tidak terduga |

---

Dokumentasi ini disusun berdasarkan komentar yang ada dalam kode serta hasil analisis struktur dan alur program. File ini merupakan komponen penting dalam sistem pembayaran yang menghubungkan Midtrans dengan database aplikasi.
