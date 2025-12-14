# Arsitektur Database & Sinkronisasi - FarmChain App

## 📋 Ringkasan
Aplikasi ini menggunakan **hybrid database architecture** dengan:
- **Room Database** (Local/Offline) - SQLite untuk penyimpanan lokal
- **Firebase Realtime Database** (Cloud/Online) - untuk sinkronisasi antar device

---

## 🗄️ Struktur Database

### 1. **Room Database (Local Storage)**

Room adalah abstraksi SQLite untuk penyimpanan data lokal di device Android.

#### **Entitas (Tables)**

##### `users` Table
```kotlin
data class UserEntity(
    @PrimaryKey val id: String,          // User ID (dari Firebase Auth)
    val name: String,                     // Nama user
    val email: String,                    // Email
    val phone: String,                    // No. HP
    val address: String,                  // Alamat
    val photoUrl: String?,                // URL foto profil
    val role: String                      // Role: "farmer", "customer", "courier"
)
```

##### `harvests` Table
```kotlin
data class HarvestEntity(
    @PrimaryKey val id: String,          // ID harvest (UUID)
    val userId: String,                   // ID petani pemilik
    val commodityType: String,            // Jenis komoditas (Padi, Jagung, dll)
    val quantity: Double,                 // Jumlah hasil panen
    val unit: String,                     // Satuan (kg, ton, dll)
    val quality: String,                  // Kualitas (Baik, Sedang, Buruk)
    val harvestDate: Long,                // Tanggal panen (timestamp)
    val photoUrl: String?,                // URL foto hasil panen
    val status: String                    // Status: available, sold, deleted
)
```

##### `requests` (Market Requests) Table
```kotlin
data class MarketRequestEntity(
    @PrimaryKey val id: String,          // ID pesanan (UUID)
    val userId: String,                   // ID customer/pembeli
    val commodityType: String,            // Jenis komoditas yang diminta
    val quantityNeeded: Double,           // Jumlah yang dibutuhkan
    val unit: String,                     // Satuan
    val location: String,                 // Lokasi pengiriman
    val deadline: Long,                   // Batas waktu (timestamp)
    val requestDate: Long,                // Tanggal pesanan (timestamp)
    val status: String,                   // Status pesanan (lihat flow di bawah)
    val harvestId: String?,               // ID harvest yang dipilih
    val farmerId: String?,                // ID petani yang memproses
    val rating: Int?                      // Rating 1-5 setelah selesai
)
```

##### `distributions` Table
```kotlin
data class DistributionEntity(
    @PrimaryKey val id: String,          // ID distribusi
    val harvestId: String,                // ID harvest
    val requestId: String,                // ID request
    val courierId: String,                // ID kurir
    val scheduledDate: Long,              // Jadwal pengiriman
    val status: String,                   // Status distribusi
    val route: String?,                   // Rute pengiriman
    val proofPhotoUrl: String?            // Foto bukti pengiriman
)
```

##### `tracking_logs` Table
```kotlin
data class TrackingLogEntity(
    @PrimaryKey val id: String,          // ID tracking
    val distributionId: String,           // ID distribusi
    val latitude: Double,                 // Latitude GPS
    val longitude: Double,                // Longitude GPS
    val timestamp: Long,                  // Waktu tracking
    val description: String               // Deskripsi lokasi
)
```

##### `notifications` Table
```kotlin
data class NotificationEntity(
    @PrimaryKey val id: String,          // ID notifikasi
    val userId: String,                   // ID user penerima
    val title: String,                    // Judul notifikasi
    val message: String,                  // Isi notifikasi
    val timestamp: Long,                  // Waktu notifikasi
    val isRead: Boolean                   // Sudah dibaca atau belum
)
```

---

### 2. **Firebase Realtime Database (Cloud Storage)**

Struktur data di Firebase mirror dengan Room, tapi dalam format JSON:

```
farmchain/
├── users/
│   ├── {userId}/
│   │   ├── name: "John Doe"
│   │   ├── email: "john@example.com"
│   │   ├── phone: "08123456789"
│   │   ├── address: "Jl. Merdeka No. 1"
│   │   ├── photoUrl: "https://..."
│   │   └── role: "farmer"
│   
├── harvests/
│   ├── {harvestId}/
│   │   ├── userId: "farmer123"
│   │   ├── commodityType: "Padi"
│   │   ├── quantity: 1000.0
│   │   ├── unit: "kg"
│   │   ├── quality: "Baik"
│   │   ├── harvestDate: 1702540800000
│   │   ├── photoUrl: "https://..."
│   │   └── status: "available"
│   
├── requests/
│   ├── {requestId}/
│   │   ├── userId: "customer456"
│   │   ├── commodityType: "Padi"
│   │   ├── quantityNeeded: 100.0
│   │   ├── unit: "kg"
│   │   ├── location: "Jakarta"
│   │   ├── deadline: 1703145600000
│   │   ├── requestDate: 1702540800000
│   │   ├── status: "pending"
│   │   ├── harvestId: "harvest789"
│   │   ├── farmerId: "farmer123"
│   │   └── rating: null
│   
├── distributions/
│   └── ...
│   
├── tracking_logs/
│   └── ...
│   
└── notifications/
    └── ...
```

---

## 🔄 Flow Sinkronisasi Data

### **Write Operation (Simpan Data)**
```
User Action → Repository → Room Database (Save) → Firebase (Sync)
                               ↓ (Immediate)      ↓ (Background)
                          Update UI           Cloud Storage
```

1. Data **langsung disimpan ke Room** untuk response cepat
2. UI **langsung update** dari Room (offline-first)
3. Background sync ke **Firebase** untuk sinkronisasi cloud
4. Jika gagal (offline), akan di-sync otomatis saat online

### **Read Operation (Baca Data)**
```
App Start → Room Database → UI Display
              ↓
        Firebase Listener (Real-time updates)
              ↓
        Update Room when data changes
              ↓
        UI auto-updates (Flow)
```

---

## 📊 Status Flow Pesanan

### **Status dalam `MarketRequestEntity`**

```
1. pending          → Pesanan baru, menunggu konfirmasi petani
2. processing       → Petani menerima, sedang menyiapkan pesanan
3. shipping         → Pesanan dikirim ke pelanggan
4. delivered        → Pesanan sudah sampai
5. cancelled        → Pesanan dibatalkan
```

### **Flow untuk Petani (Farmer)**
```
pending → [Klik "Terima Pesanan"] → processing
          ↓
processing → [Klik "Kirim Pesanan"] → shipping
             ↓
shipping → [Klik "Konfirmasi Sampai"] → delivered
```

### **Flow untuk Pelanggan (Customer)**
```
pending → Menunggu konfirmasi petani
processing → Pesanan sedang disiapkan
shipping → Pesanan dalam pengiriman
delivered → [Dapat memberikan rating] → Selesai
```

### **Tab di Farmer Screen**
- **Tab "Aktif"**: Menampilkan pesanan dengan status `pending`, `processing`, `shipping`
- **Tab "Selesai"**: Menampilkan pesanan dengan status `delivered`
- **Tab "Batal"**: Menampilkan pesanan dengan status `cancelled`

---

## 🔧 Komponen Utama

### **1. DAO (Data Access Object)**
File: `app/src/main/java/com/farmchain/data/local/dao/`

Berisi interface untuk akses database Room:
- `UserDao.kt` - CRUD operations untuk users
- `HarvestDao.kt` - CRUD operations untuk harvests
- `MarketRequestDao.kt` - CRUD operations untuk market requests
- dll.

### **2. Repository**
File: `app/src/main/java/com/farmchain/data/repository/`

Layer business logic yang menghubungkan DAO (Room) dengan Firebase:
- `UserRepository.kt` - Manage user data
- `MarketRequestRepository.kt` - Manage order/request data
- dll.

### **3. Firebase Data Source**
File: `app/src/main/java/com/farmchain/data/remote/datasource/FirebaseDataSource.kt`

Menangani semua operasi Firebase Realtime Database.

### **4. Sync Manager**
File: `app/src/main/java/com/farmchain/data/sync/FirebaseSyncManager.kt`

Menangani sinkronisasi otomatis antara Room dan Firebase.

---

## 🚀 Cara Kerja Real-time Updates

### **Firebase Listeners**
```kotlin
// Contoh: Listen perubahan pada requests
database.child("requests")
    .addValueEventListener(object : ValueEventListener {
        override fun onDataChange(snapshot: DataSnapshot) {
            // Update Room database
            // UI akan otomatis update via Flow
        }
    })
```

### **Room Flow**
```kotlin
// Contoh: Observe data dari Room
@Query("SELECT * FROM requests WHERE userId = :userId")
fun observeRequests(userId: String): Flow<List<MarketRequestEntity>>
```

UI akan **otomatis update** ketika data berubah (reactive programming).

---

## ⚡ Keunggulan Arsitektur Ini

1. **Offline-First**: App tetap berfungsi tanpa internet
2. **Real-time Sync**: Perubahan langsung tersinkronisasi ke semua device
3. **Performance**: Room cache lokal membuat app cepat
4. **Reliable**: Data tidak hilang karena disimpan lokal dan cloud
5. **Scalable**: Mudah menambah fitur baru

---

## 🔐 Firebase Security Rules

File: `database.rules.json`

Rules ini mengatur siapa yang bisa read/write data:
```json
{
  "rules": {
    "users": {
      "$uid": {
        ".read": "$uid === auth.uid || auth != null",
        ".write": "$uid === auth.uid"
      }
    },
    "harvests": {
      ".read": "auth != null",
      ".write": "auth != null"
    },
    "requests": {
      ".read": "auth != null",
      ".write": "auth != null"
    }
  }
}
```

**Penjelasan:**
- `users/{uid}`: User hanya bisa edit data sendiri
- `harvests` & `requests`: Semua user login bisa read/write
- **Auth required**: Semua operasi butuh user login

---

## 📝 File `database.rules.json`

**Bisa dihapus** setelah rules di-upload ke Firebase Console.

File ini **hanya untuk referensi** atau deployment otomatis. Rules yang aktif ada di Firebase Console.

### Upload Rules ke Firebase:
1. Buka Firebase Console → Realtime Database → Rules
2. Copy-paste isi `database.rules.json`
3. Klik "Publish"

---

## 🎯 Kesimpulan

- **Room** = Database lokal di HP (cepat, offline)
- **Firebase** = Database cloud (sync antar device, real-time)
- **Repository** = Jembatan antara Room & Firebase
- **Flow** = Reactive updates untuk UI

Semua berjalan otomatis di background! 🚀

---

## 🐛 Troubleshooting

### "Data tidak sinkron"
✅ Pastikan device terkoneksi internet
✅ Cek Firebase Rules sudah benar
✅ Verify `google-services.json` sudah benar

### "Pesanan hilang setelah konfirmasi"
✅ Sudah diperbaiki: Tab "Aktif" sekarang menampilkan semua status kecuali delivered & cancelled
✅ Gunakan tab "Selesai" untuk melihat pesanan yang sudah selesai

### "Login masuk ke akun lain"
✅ Pastikan SessionManager menyimpan userId yang benar
✅ Clear app data dan login ulang

---

*Last Updated: December 14, 2025*
*Version: 1.0*

