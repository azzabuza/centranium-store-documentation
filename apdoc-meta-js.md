## **1. Deskripsi Umum**
File ini merupakan **handler API endpoint** yang berfungsi untuk **menghasilkan halaman HTML dengan Open Graph meta tags** untuk keperluan **social media sharing**. File ini mendeteksi apakah pengunjung adalah bot sosial media (Facebook, WhatsApp, Twitter, Telegram, Discord) atau manusia, lalu memberikan respons yang sesuai.

---

## **2. Fungsi Utama**
```javascript
export default async function handler(req, res) { ... }
```
**Handler utama** yang mengekspor fungsi untuk menangani request meta data produk.

---

## **3. Fitur-Fitur dan Penjelasan Kode**

### **3.1 Ekstraksi Parameter**
```javascript
const { product: productPermalink } = req.query;
const userAgent = req.headers['user-agent'] || '';
```
**Fitur:** Mengambil parameter dari URL dan user agent.  
**Penjelasan:**  
- `productPermalink`: Slug/nama unik produk yang diambil dari query string
- `userAgent`: Identitas browser atau bot pengunjung

---

### **3.2 Bot Detection**
```javascript
const isBot = /facebookexternalhit|WhatsApp|Twitterbot|TelegramBot|Discordbot|Embedly/i.test(userAgent);
```
**Fitur:** Mendeteksi apakah pengunjung adalah bot social media.  
**Penjelasan:** Regex pattern untuk mendeteksi bot dari platform:
- `facebookexternalhit` - Facebook crawler
- `WhatsApp` - WhatsApp crawler  
- `Twitterbot` - Twitter crawler
- `TelegramBot` - Telegram crawler
- `Discordbot` - Discord crawler
- `Embedly` - Embedly crawler

---

### **3.3 Logging Visitor**
```javascript
console.log(`[API START] Visitor: ${isBot ? 'BOT' : 'HUMAN'} (${userAgent}) | Product: "${productPermalink}"`);
```
**Fitur:** Mencatat informasi pengunjung.  
**Penjelasan:** Membedakan log antara bot dan manusia untuk keperluan debugging dan analisis.

---

### **3.4 Validasi Product Permalink**
```javascript
if (!productPermalink) return res.redirect('/');
```
**Fitur:** Validasi parameter produk.  
**Penjelasan:** Jika tidak ada parameter produk, redirect ke halaman utama.

---

### **3.5 Firebase Configuration**

#### **Konfigurasi Kredensial**
```javascript
const FIREBASE_SECRET = "F2BadnChIXVM700KUJqsVOd7EKJM5Tw2CdDDvQVD";
const BASE_DB_URL = "https://centranium-store-default-rtdb.asia-southeast1.firebasedatabase.app/products.json";
const DB_URL = `${BASE_DB_URL}?auth=${FIREBASE_SECRET}`;
```
**Fitur:** Konfigurasi koneksi ke Firebase.  
**Penjelasan:**
- Menggunakan **Firebase Secret** untuk autentikasi (legacy)
- Mengakses endpoint `products.json` langsung
- **Catatan Keamanan:** Firebase Secret adalah metode autentikasi lama. Sebaiknya diganti dengan token autentikasi yang lebih aman.

---

### **3.6 Fetch Products dari Firebase**
```javascript
const response = await fetch(DB_URL, {
  method: 'GET',
  headers: {
    'Accept': 'application/json',
    'User-Agent': 'Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)'
  }
});
```
**Fitur:** Mengambil data produk dari Firebase.  
**Penjelasan:**
- Menggunakan method `GET`
- **User-Agent** disamarkan sebagai Googlebot untuk menghindari pemblokiran
- Mengambil seluruh data produk dalam satu request

---

### **3.7 Error Handling Firebase**
```javascript
if (!response.ok) {
    const errText = await response.text();
    console.error(`[FIREBASE ERROR] Status: ${response.status} - ${errText}`);
    throw new Error(`Firebase Error: ${response.status}`);
}
```
**Fitur:** Menangani error dari Firebase.  
**Penjelasan:** Mencatat status error dan pesan dari respons Firebase.

---

### **3.8 Parsing Data Produk**
```javascript
const shoppings = await response.json();
if (!shoppings) return res.redirect('/404.html');
```
**Fitur:** Mengkonversi respons JSON dan validasi data.  
**Penjelasan:** Jika data produk kosong, redirect ke halaman 404.

---

### **3.9 Pencarian Produk berdasarkan Permalink**
```javascript
let product = null;
const targetLink = String(productPermalink).trim();

for (const key in shoppings) {
  if (shoppings[key] && shoppings[key].permalink === targetLink) {
    product = shoppings[key];
    break;
  }
}
```
**Fitur:** Mencari produk spesifik berdasarkan permalink.  
**Penjelasan:**
- Melakukan iterasi semua produk di database
- Membandingkan `permalink` dengan parameter URL
- Mengambil produk pertama yang cocok

---

### **3.10 Validasi Produk Ditemukan**
```javascript
if (!product) {
  console.log(`[API FAIL] Produk tidak ditemukan: ${targetLink}`);
  if (isBot) return res.status(404).send("<h1>Produk tidak ditemukan</h1>");
  return res.redirect('/404.html');
}
```
**Fitur:** Menangani produk tidak ditemukan.  
**Penjelasan:**
- **Bot:** Menampilkan halaman error 404 sederhana
- **Manusia:** Redirect ke halaman 404 HTML

---

### **3.11 Pengumpulan Meta Data**

#### **Title**
```javascript
const title = product.title ? product.title : "Produk Centranium";
```
**Fitur:** Mengambil judul produk.

---

#### **Image**
```javascript
const image = product.image1 || "https://centranium.com/asset/image/logo.webp";
```
**Fitur:** Mengambil gambar produk.  
**Penjelasan:** Menggunakan `image1` sebagai gambar utama, fallback ke logo default.

---

#### **Original URL**
```javascript
const originalUrl = `https://www.centranium.com/product/shopping.html?product=${encodeURIComponent(targetLink)}`;
```
**Fitur:** Membangun URL asli produk.  
**Penjelasan:** URL yang akan dituju setelah meta tags diambil.

---

#### **Deskripsi**
```javascript
let descriptionText = "Beli produk digital terbaik di Centranium.";
if (Array.isArray(product.description)) {
  descriptionText = product.description.join(" ");
} else if (typeof product.description === 'string') {
  descriptionText = product.description;
}
descriptionText = descriptionText.replace(/<[^>]*>?/gm, '').substring(0, 160) + "...";
```
**Fitur:** Memproses deskripsi produk.  
**Penjelasan:**
- Mendukung deskripsi dalam format **Array** atau **String**
- Menghapus **tag HTML** dari deskripsi
- Membatasi panjang **160 karakter** (optimal untuk meta description)
- Menambahkan ellipsis di akhir

---

#### **Harga**
```javascript
let priceText = "";
if (product.price) {
    try {
        const rawPrice = product.price.toString().replace(/\./g, "").replace(/,/g, ".");
        const priceVal = parseFloat(rawPrice);
        if (!isNaN(priceVal)) priceText = "Rp " + priceVal.toLocaleString("id-ID");
    } catch (e) {}
}
```
**Fitur:** Memformat harga produk.  
**Penjelasan:**
- Membersihkan format angka (menghapus titik sebagai pemisah ribuan)
- Mengganti koma dengan titik untuk desimal
- Memformat ke mata uang Rupiah (Rp)
- Menggunakan locale Indonesia (id-ID)

---

#### **Full Description**
```javascript
const fullDescription = priceText ? `Harga: ${priceText}. ${descriptionText}` : descriptionText;
```
**Fitur:** Menggabungkan harga dan deskripsi.  
**Penjelasan:** Menambahkan informasi harga di awal deskripsi jika tersedia.

---

### **3.12 Redirect Script**
```javascript
const redirectScript = isBot ? '' : `window.location.replace("${originalUrl}");`;
```
**Fitur:** Membuat script redirect.  
**Penjelasan:**
- **Bot:** Tidak perlu redirect (bot hanya membaca meta tags)
- **Manusia:** Redirect otomatis ke halaman produk

---

### **3.13 HTML Response Builder**

#### **Meta Tags untuk Social Media**

**Open Graph Protocol (Facebook, WhatsApp, LinkedIn):**
```html
<meta property="og:type" content="website">
<meta property="og:title" content="${title}">
<meta property="og:description" content="${fullDescription}">
<meta property="og:image" content="${image}">
<meta property="og:image:width" content="600">
<meta property="og:image:height" content="600">
<meta property="og:site_name" content="Centranium Store">
```

**Twitter Card:**
```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="${title}">
<meta name="twitter:description" content="${fullDescription}">
<meta name="twitter:image" content="${image}">
```

**Meta Standar:**
```html
<title>${title}</title>
<meta name="description" content="${fullDescription}">
```

---

#### **Font dan Styling**
```html
<link href="https://fonts.googleapis.com/css2?family=Inter:ital,wght@0,100..900;1,100..900&display=swap" rel="stylesheet">
```
**Fitur:** Mengintegrasikan Google Fonts.  
**Penjelasan:** Menggunakan font **Inter** untuk tampilan yang modern dan konsisten.

---

#### **JavaScript Redirect**
```html
<script>
  ${redirectScript}
</script>
```
**Fitur:** Redirect otomatis untuk pengunjung manusia.

---

#### **Fallback Tampilan**
```html
<body style="font-family: 'Inter', sans-serif;">
  <div style="text-align: center; padding: 50px;">
    <img src="${image}">
    <h1>${title}</h1>
    <p>${fullDescription}</p>
    <a href="${originalUrl}">Lihat Detail Produk</a>
  </div>
</body>
```
**Fitur:** Tampilan fallback jika JavaScript tidak berjalan.  
**Penjelasan:** Menampilkan informasi produk dengan tombol link manual.

---

### **3.14 Response Headers**
```javascript
res.setHeader('Content-Type', 'text/html; charset=utf-8');
res.setHeader('Cache-Control', 's-maxage=10, stale-while-revalidate=59');
```
**Fitur:** Mengatur header respons.  
**Penjelasan:**
- `Content-Type`: Mengembalikan HTML dengan encoding UTF-8
- `Cache-Control`: 
  - `s-maxage=10`: Cache di server selama 10 detik
  - `stale-while-revalidate=59`: Boleh menggunakan cache kadaluarsa selama 59 detik sambil revalidate

---

### **3.15 Error Handling Global**
```javascript
catch (error) {
    console.error("[API ERROR]", error);
    if (isBot) return res.status(500).send(`Server Error: ${error.message}`);
    const fallbackUrl = `https://www.centranium.com/product/shopping.html?product=${productPermalink}`;
    return res.redirect(fallbackUrl);
}
```
**Fitur:** Menangani error tidak terduga.  
**Penjelasan:**
- **Bot:** Menampilkan pesan error server
- **Manusia:** Redirect ke halaman produk langsung

---

## **4. Alur Kerja Program**

```
[USER/BOT]
    ↓ (Request dengan ?product=...)
[BOT DETECTION]
    ├── Facebook crawler
    ├── WhatsApp crawler
    ├── Twitterbot
    ├── TelegramBot
    ├── Discordbot
    └── Embedly
    ↓
[VALIDASI PERMALINK]
    ↓
[FETCH DATA FIREBASE]
    ↓
[PARSING & SEARCH PRODUK]
    ↓
[PRODUK DITEMUKAN?]
    ├── Ya   → Generate Meta Tags
    └── Tidak→ 404 (Bot: text, Human: redirect)
    ↓
[GENERATE HTML]
    ├── Meta Tags (OG & Twitter)
    ├── Redirect Script (Human only)
    └── Fallback UI
    ↓
[RETURN RESPONSE]
```

---

## **5. Struktur Data Produk yang Diharapkan**

```json
{
  "productKey": {
    "permalink": "nama-produk-unik",
    "title": "Judul Produk",
    "description": "Deskripsi produk..." atau ["Array", "deskripsi"],
    "image1": "https://url-gambar-produk.jpg",
    "price": "100.000" atau 100000
  }
}
```

---

## **6. Fitur Keamanan & Optimasi**

### **Sudah Diimplementasikan:**
✅ Deteksi bot vs manusia  
✅ Sanitasi HTML dari deskripsi  
✅ Redirect untuk pengunjung manusia  
✅ Cache control untuk performa  
✅ Error handling komprehensif  
✅ Fallback value untuk semua field  

### **Potensi Pengembangan:**
🔐 **Ganti Firebase Secret** → Gunakan Firebase Admin SDK  
🔐 **Validasi Input** → Sanitasi productPermalink  
🔐 **Rate Limiting** → Cegah abuse API  
🔐 **Image Optimization** → Compress/resize gambar untuk meta tags  
🔐 **Structured Data** → Tambahkan JSON-LD untuk rich snippets  

---

## **7. Social Media Platform yang Didukung**

| Platform | Bot Pattern | Meta Tags Digunakan |
|----------|------------|---------------------|
| Facebook | `facebookexternalhit` | Open Graph |
| WhatsApp | `WhatsApp` | Open Graph |
| Twitter | `Twitterbot` | Twitter Cards |
| Telegram | `TelegramBot` | Open Graph |
| Discord | `Discordbot` | Open Graph |
| Embedly | `Embedly` | Open Graph |

---

## **8. Logging Events**

| Event | Level | Keterangan |
|-------|-------|------------|
| `[API START] Visitor: BOT/HUMAN` | Info | Mencatat jenis pengunjung dan produk |
| `[FIREBASE ERROR]` | Error | Gagal mengambil data dari Firebase |
| `[API FAIL] Produk tidak ditemukan` | Warning | Produk dengan permalink tidak ada |
| `[API ERROR]` | Error | Error tidak terduga |

---

## **9. Performance Considerations**

### **Cache Strategy:**
- **s-maxage=10**: CDN/Edge cache selama 10 detik
- **stale-while-revalidate=59**: Bisa serve cache lama selama revalidate

### **Optimasi:**
- Data produk di-fetch dari Firebase setiap request (perlu caching)
- Meta tags di-generate dinamis
- Font Google di-load terpisah

---

Dokumentasi ini disusun berdasarkan komentar yang ada dalam kode serta hasil analisis struktur dan alur program. File `meta.js` merupakan komponen penting untuk **Social Media SEO** yang memastikan setiap produk memiliki tampilan yang menarik ketika dibagikan di platform sosial media.
