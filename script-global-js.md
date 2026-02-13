## 📋 **Ikhtisar Program**
File `global.js` merupakan komponen JavaScript utama yang menangani berbagai fungsionalitas global untuk website Centranium Digital Store (CDS). Program ini bertanggung jawab atas manajemen komponen UI, autentikasi pengguna, pencarian produk, dan berbagai fitur interaktif lainnya.

---

## 🔧 **FITUR-FITUR UTAMA**

### 1. **WIDGET - Komponen UI Dinamis**
Program mengambil elemen-elemen utama DOM yang akan digunakan secara global:

```javascript
const header = document.getElementById("header");
const footer = document.getElementById("footer");
const mobile = document.getElementById("mobile");
```

**Fungsi:** Menyimpan referensi elemen header, footer, dan mobile untuk dimanipulasi secara dinamis.

---

### 2. **LIVE SEARCH - Pencarian Produk Real-time**
**Tujuan:** Memungkinkan pengguna mencari produk secara real-time dengan respons cepat.

#### 🔍 **Komponen Pencarian:**
- **Variabel `allProducts`**: Array global yang menyimpan semua produk untuk keperluan pencarian
- **Fungsi `showSearchResults(query)`**: Menampilkan hasil pencarian berdasarkan input pengguna
- **Trigger Event**: 
  - `input` → Menampilkan hasil saat mengetik
  - `focus` → Menampilkan hasil saat input difokuskan
  - `click` di luar area → Menyembunyikan hasil

**Fitur Pendukung:**
- Validasi panjang query minimal 2 karakter
- Pencarian case-insensitive (menggunakan `toUpperCase()`)
- Tampilan "Produk Tidak Ditemukan" dengan call-to-action
- Tombol cancel untuk menghapus input dan menutup hasil

---

### 3. **CLEAN AND PARSE PRICE - Pengolahan Format Harga**
**Fungsi:** `cleanAndParsePrice(price)`

**Tujuan:** Membersihkan format harga Indonesia dan mengkonversinya ke format numerik.

**Proses:**
1. Menangani nilai undefined/null/kosong
2. Menghapus titik (.) sebagai pemisah ribuan
3. Mengganti koma (,) dengan titik sebagai pemisah desimal
4. Mengkonversi ke tipe data float
5. Fallback ke 0 jika parsing gagal

**Contoh:** 
- `"Rp 1.500.000,50"` → `1500000.50`
- `"100.000"` → `100000`

---

### 4. **FETCH ALL PRODUCTS - Pengambilan Data Produk**
**Fungsi:** `fetchAllProducts()`

**Tujuan:** Mengambil seluruh data produk dari Firebase Realtime Database untuk kebutuhan pencarian dan SEO.

#### 📊 **Alur Kerja:**
1. Cek apakah data sudah pernah diambil (`allProducts.length > 0`)
2. Verifikasi ketersediaan koneksi Firebase
3. Ambil data dari path `"products"` di database
4. Konversi data ke format array
5. Simpan ke variabel global `allProducts`

#### 🔍 **Pencegahan Duplikasi:**
```javascript
if (allProducts.length > 0) {
    return; // Hentikan eksekusi jika data sudah ada
}
```

---

### 5. **INSERT JSON-LD - Structured Data SEO**
**Tujuan:** Menambahkan markup Schema.org untuk meningkatkan visibilitas di mesin pencari.

#### 🏷️ **Jenis Data Terstruktur:**
- **@type**: "Product" - Menandai konten sebagai produk
- **Informasi yang disertakan:**
  - Nama produk
  - Gambar
  - Deskripsi
  - Penawaran (harga, mata uang, ketersediaan)
  - Merek
  - Kebijakan pengembalian

**Implementasi:** Script JSON-LD disisipkan ke `<head>` dokumen HTML.

---

### 6. **RENDER PRODUCT - Templating Produk**
**Fungsi:** Membuat HTML untuk setiap produk hasil pencarian.

**Elemen yang Dirender:**
- Gambar produk (dengan lazy loading)
- Judul produk
- Harga (dengan format Rupiah)
- Harga asli (jika ada diskon)
- Persentase diskon (jika ada)
- Deskripsi (meta tag)

**Atribut Schema.org yang Ditambahkan:**
- `itemscope`, `itemtype`, `itemprop` untuk markup data terstruktur

---

### 7. **AUTHENTICATION SYSTEM - Sistem Autentikasi**

#### 👤 **UPDATE USER NAME DISPLAY**
**Fungsi:** `updateUserNameDisplay()`

**Tujuan:** Memperbarui tampilan nama pengguna di header.

**Strategi Caching Bertingkat:**

1️⃣ **Cache Check (Prioritas Tertinggi)**
```javascript
const cachedName = localStorage.getItem(`userName_${user.uid}`);
if (cachedName) {
    displayUserNameFull(cachedName, userNameElement);
}
```
- Mengambil nama dari localStorage untuk respons cepat
- Menggunakan UID sebagai key unik

2️⃣ **Database Fetch (Sumber Kebenaran)**
```javascript
fetchUserDataFromDatabase(user.uid)
    .then(userData => {
        if (userData && userData.name) {
            localStorage.setItem(`userName_${user.uid}`, userData.name);
            displayUserNameFull(userData.name, userNameElement);
        }
    })
```
- Mengambil data terbaru dari Firebase
- Memperbarui cache jika ada perubahan

3️⃣ **Fallback to Auth (Cadangan)**
```javascript
function fallbackToAuthData(user, userNameElement)
```
- Menggunakan `displayName` dari Firebase Auth
- Jika tidak ada, gunakan bagian sebelum '@' dari email

#### 🔄 **REAL-TIME LISTENER**
**Fungsi:** Memantau perubahan data pengguna secara real-time.

```javascript
userRef.on('value', (snapshot) => { ... })
```
- Mendengarkan perubahan di path `users/{userId}`
- Memperbarui tampilan dan cache secara otomatis

#### 🚫 **LOGOUT RESET**
- Mengembalikan teks menjadi "Akun" saat logout
- Menghapus atribut `data-fullname`
- Membersihkan cache localStorage

---

### 8. **SHOPPING CART - Manajemen Keranjang**

#### 🛒 **UPDATE CART COUNT**
**Fungsi:** `updateCartCount()`

**Tujuan:** Menampilkan jumlah item dalam keranjang.

**Alur Kerja:**
1. Cek status autentikasi pengguna
2. Jika tidak login → tampilkan "0"
3. Jika login → ambil data dari path `carts/{userId}`
4. Hitung total kuantitas dengan `reduce()`
5. Perbarui elemen `#cart-count`

**Penanganan Error:**
- Try-catch untuk operasi async
- Fallback ke "0" jika terjadi kesalahan

---

### 9. **LOCKDOWN SYSTEM - Sistem Pemblokiran Akun**

#### 🔒 **CHECK LOCKDOWN STATUS**
**Fungsi:** `async function checkUserLockdownStatus(user)`

**Tujuan:** Memeriksa apakah akun pengguna diblokir oleh administrator.

**Logika:**
```javascript
return userData.locked !== true; // true = tidak diblokir, false = diblokir
```

#### 🚨 **LOCKDOWN MONITOR**
**Fungsi:** Memantau dan menangani akun yang diblokir.

**Alur:**
1. Setiap perubahan status autentikasi
2. Periksa status lockdown
3. Jika diblokir → logout paksa
4. Tampilkan alert peringatan
5. Redirect ke halaman login

**Pesan Alert:**
```
❌ AKUN DINONAKTIFKAN
Akun Anda (email) telah dinonaktifkan sementara oleh Administrator.
Hubungi Administrator untuk informasi lebih lanjut.
```

---

### 10. **ONLINE STATUS - Status Kehadiran Pengguna**

#### 💤 **BEFORE UNLOAD**
**Fungsi:** Event listener `beforeunload`

**Tujuan:** Memperbarui status pengguna saat meninggalkan halaman.

**Update Data:**
- `online: false` - Menandai pengguna offline
- `lastSeen: TIMESTAMP` - Mencatat waktu terakhir aktif

**Manfaat:**
- Menampilkan status online/offline yang akurat
- Melacak aktivitas pengguna
- Fitur "last seen" seperti aplikasi chat

---

### 11. **WELCOME POPUP - Popup Selamat Datang**

**Tujuan:** Menampilkan sambutan untuk pengunjung pertama kali.

#### 🎯 **Fitur:**
- **Trigger:** Hanya sekali per browser (menggunakan localStorage)
- **Konten:** Sambutan dan informasi tentang CDS
- **CTA:** Tombol "Jangan Tampilkan Lagi"
- **Close Option:** Klik tombol atau area luar popup

**Persistensi:**
```javascript
localStorage.setItem('popupShown', 'true');
```

---

### 12. **BETA BADGE - Lencana Pengembangan**

**Tujuan:** Menandai website dalam tahap pengembangan (Beta).

**Karakteristik:**
- Posisi: Fixed di sudut kiri atas
- Rotasi: -45 derajat
- Warna: Merah dengan transparansi (RGB 255,71,87, 0.8)
- Z-index: 9999 (paling atas)
- Teks: "BETA" dengan letter spacing

**Pencegahan Duplikasi:**
```javascript
if (!document.querySelector('.beta')) { ... }
```

---

### 13. **COMPONENT LOADING - Pemuatan Komponen**

#### 📦 **Header, Footer, dan Mobile Components**
**Fungsi:** Memuat komponen HTML secara asinkron.

**Alur:**
1. Fetch file HTML dari direktori `../component/`
2. Konversi response ke teks
3. Sisipkan ke elemen DOM yang sesuai

**Inisialisasi di Header:**
Setelah header dimuat, sistem menginisialisasi:
- Nama pengguna dari cache
- Jumlah keranjang
- Nama pengguna dari database
- Data produk untuk pencarian
- Event listener pencarian

---

### 14. **EXPORT FUNCTION - Fungsi Global**

**Fungsi:** `window.checkLockdown = checkUserLockdownStatus`

**Tujuan:** Mengekspos fungsi ke lingkup global untuk digunakan di file JavaScript lain.

**Penggunaan:**
```javascript
// Di file lain
const isAllowed = await window.checkLockdown(currentUser);
```

---

## 📊 **FLOWCHART PROGRAM**

```
START
    ↓
├── LOAD COMPONENTS
│   ├── header.html → Inisialisasi fitur header
│   ├── footer.html → Render footer
│   └── mobile.html → Render mobile view
    ↓
├── AUTH STATE MONITOR
│   ├── LOGIN → 
│   │   ├── Update nama pengguna (cache + DB)
│   │   ├── Update jumlah keranjang
│   │   ├── Cek status lockdown
│   │   └── Set real-time listener
│   └── LOGOUT →
│       ├── Reset UI
│       └── Hapus cache
    ↓
├── LIVE SEARCH
│   ├── Input event → Filter produk → Render hasil
│   ├── Focus event → Tampilkan hasil (jika ada)
│   └── Cancel button → Reset pencarian
    ↓
├── BETA BADGE → Tampilkan lencana Beta
    ↓
├── WELCOME POPUP → Tampilkan (jika pertama kali)
    ↓
└── BEFORE UNLOAD → Update status online
    ↓
END
```

---

## 🔐 **MANAJEMEN CACHE DAN STORAGE**

### **localStorage Keys:**
| Key Pattern | Deskripsi | Contoh |
|------------|----------|--------|
| `userName_{uid}` | Nama pengguna yang sudah login | `userName_abc123` |
| `popupShown` | Status popup selamat datang | `true` |

### **Strategi Caching:**
1. **Read**: Prioritaskan cache untuk respons cepat
2. **Write**: Perbarui cache saat data berubah
3. **Invalidation**: Hapus cache saat logout

---

## ⚠️ **ERROR HANDLING**

Program menerapkan penanganan error di berbagai level:

1. **Promise-level**: `.catch()` untuk operasi async
2. **Function-level**: Try-catch untuk fungsi async
3. **Value-level**: Validasi input dan fallback values
4. **Resource-level**: Pengecekan eksistensi elemen DOM

**Contoh:**
```javascript
if (!productListContainer || !searchResultContainer) return;
if (price === undefined || price === null || price === "") return 0;
if (typeof db !== 'undefined') { ... } else { console.error(...) }
```

---

## 📱 **RESPONSIVITAS DAN AKSESIBILITAS**

1. **Lazy Loading Images**: Atribut `loading="lazy"` pada gambar
2. **Semantic HTML**: Penggunaan `itemprop` dan `itemscope`
3. **ARIA-ready**: Struktur HTML yang mendukung screen reader
4. **Mobile Component**: Komponen khusus untuk tampilan mobile

---

## 🚀 **OPTIMASI PERFORMANCE**

1. **Data Caching**: Menyimpan data produk dan nama pengguna
2. **Conditional Fetching**: Hanya ambil data jika belum ada
3. **Debouncing**: (Implisit) Pencarian dipicu per input event
4. **Minimal DOM Manipulation**: Update hanya elemen yang diperlukan
5. **Asynchronous Loading**: Fetch komponen tanpa blocking

---

## 🧩 **DEPENDENCIES**

1. **Firebase** (Global object):
   - `firebase.auth()` - Autentikasi
   - `firebase.database()` - Realtime Database
2. **HTML Components**:
   - `header.html` - Navigasi dan pencarian
   - `footer.html` - Informasi footer
   - `mobile.html` - Tampilan mobile

---

Dokumentasi ini mencakup seluruh fitur yang diimplementasikan dalam file `global.js` berdasarkan analisis kode dan komentar yang tersedia. Program menunjukkan arsitektur yang terstruktur dengan pemisahan tanggung jawab yang jelas dan strategi caching yang efektif.
