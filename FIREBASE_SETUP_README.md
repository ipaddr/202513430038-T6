# Firebase Realtime Database Setup Guide

## Overview
Aplikasi ini menggunakan Firebase Realtime Database sebagai cloud database dan Room sebagai local database untuk offline-first architecture.

## Arsitektur Data

### 1. Firebase Realtime Database (Cloud)
- Database utama yang di-host di Firebase
- Real-time synchronization
- Offline persistence enabled
- Security rules untuk data protection

### 2. Room Database (Local)
- Cache lokal untuk offline access
- Sync otomatis dengan Firebase
- Faster query performance
- Works without internet connection

### 3. Data Flow
```
User Action → Repository → Firebase Database (write)
                         ↓
                    Room Database (cache)
                         ↓
                    LiveData/Flow → UI
```

## File-file yang Dibuat

### 1. Models & Mappers
- `data/remote/model/FirebaseModels.kt` - Data models untuk Firebase
- `data/mapper/EntityMapper.kt` - Converter antara Firebase & Room entities

### 2. Data Sources
- `data/remote/datasource/FirebaseDataSource.kt` - Firebase CRUD operations

### 3. Sync Manager
- `data/sync/FirebaseSyncManager.kt` - Auto-sync Firebase ↔ Room

### 4. Repositories (Updated/Created)
- `data/repository/UserRepository.kt` - Updated untuk Firebase Realtime DB
- `data/repository/HarvestRepository.kt` - Harvest management
- `data/repository/MarketRequestRepository.kt` - Market request management
- `data/repository/DistributionRepository.kt` - Distribution management
- `data/repository/TrackingRepository.kt` - Tracking log management
- `data/repository/NotificationRepository.kt` - Notification management

### 5. Dependency Injection
- `di/FirebaseModule.kt` - Updated dengan FirebaseDatabase provider

### 6. Application Class
- `FarmChainApp.kt` - Initialize sync manager on app start

### 7. Documentation
- `DATABASE_SCHEMA.md` - Database structure documentation
- `database.rules.json` - Firebase security rules

## Setup Firebase Console

### 1. Enable Realtime Database
1. Buka [Firebase Console](https://console.firebase.google.com/)
2. Pilih project "agrologistik"
3. Klik "Realtime Database" di menu sebelah kiri
4. Klik "Create Database"
5. Pilih location (us-central1 atau asia-southeast1)
6. Start in **test mode** (untuk development)

### 2. Deploy Security Rules
1. Di Firebase Console, buka Realtime Database
2. Klik tab "Rules"
3. Copy isi dari `database.rules.json`
4. Paste dan klik "Publish"

### 3. Add Indexes (Optional untuk performance)
Di Firebase Console → Realtime Database → Rules, tambahkan indexes:
```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "harvests": {
      ".indexOn": ["userId", "status"]
    },
    "requests": {
      ".indexOn": ["userId", "status"]
    },
    "distributions": {
      ".indexOn": ["courierId", "status"]
    },
    "tracking_logs": {
      ".indexOn": ["distributionId"]
    },
    "notifications": {
      ".indexOn": ["userId", "isRead"]
    }
  }
}
```

## Cara Penggunaan

### 1. Menyimpan Data
```kotlin
// Contoh: Save harvest
val harvest = HarvestEntity(
    id = UUID.randomUUID().toString(),
    userId = currentUserId,
    commodityType = "Padi",
    quantity = 100.0,
    unit = "kg",
    quality = "Grade A",
    harvestDate = System.currentTimeMillis(),
    photoUrl = null,
    status = "AVAILABLE"
)

// Data akan otomatis tersimpan di Firebase dan Room
harvestRepository.saveHarvest(harvest)
```

### 2. Membaca Data (Real-time)
```kotlin
// Observe dari Room (auto-sync dengan Firebase)
harvestRepository.observeHarvests()
    .collect { harvests ->
        // Update UI
        updateUI(harvests)
    }
```

### 3. Sync Manual (jika diperlukan)
```kotlin
// Sync notifications untuk user tertentu
firebaseSyncManager.syncNotificationsForUser(userId)

// Sync tracking logs untuk distribution tertentu
firebaseSyncManager.syncTrackingLogsForDistribution(distributionId)
```

## Fitur-fitur

### ✅ Offline Support
- Data tetap bisa dibaca saat offline dari Room
- Perubahan akan di-sync otomatis saat online

### ✅ Real-time Updates
- Perubahan di Firebase langsung terlihat di semua devices
- FirebaseSyncManager mendengarkan perubahan real-time

### ✅ Data Persistence
- Firebase: offline persistence enabled
- Room: persistent local storage

### ✅ Type-safe Queries
- Room DAO dengan compile-time verification
- Flow-based reactive queries

## Entities & Relations

### Users
- Primary key: `id`
- Roles: FARMER, MERCHANT, COURIER

### Harvests
- Belongs to: User (farmer)
- Foreign key: `userId`
- Status: AVAILABLE, REQUESTED, SOLD

### Market Requests
- Belongs to: User (merchant)
- Foreign keys: `userId`, `harvestId`, `farmerId`
- Status: PENDING, MATCHED, COMPLETED

### Distributions
- References: Harvest, Request, User (courier)
- Foreign keys: `harvestId`, `requestId`, `courierId`
- Status: SCHEDULED, IN_TRANSIT, DELIVERED

### Tracking Logs
- Belongs to: Distribution
- Foreign key: `distributionId`
- Contains: GPS coordinates, timestamp

### Notifications
- Belongs to: User
- Foreign key: `userId`
- Filterable by: isRead status

## Testing

### Test Firebase Connection
```kotlin
// Di ViewModel atau Repository
viewModelScope.launch {
    try {
        val testUser = UserFirebase(
            id = "test123",
            name = "Test User",
            email = "test@example.com",
            phone = "08123456789",
            address = "Test Address",
            photoUrl = null,
            role = "FARMER"
        )
        firebaseDataSource.saveUser(testUser)
        println("Firebase connection OK!")
    } catch (e: Exception) {
        println("Firebase error: ${e.message}")
    }
}
```

### Monitor Firebase Data
1. Buka Firebase Console
2. Klik Realtime Database
3. Lihat tab "Data"
4. Data akan muncul real-time saat app menyimpan

## Troubleshooting

### Problem: Data tidak muncul di Firebase
**Solution:**
1. Cek internet connection
2. Verify Firebase rules allow write
3. Check Firebase project configuration di `google-services.json`

### Problem: Sync tidak bekerja
**Solution:**
1. Pastikan `FirebaseSyncManager.startSync()` dipanggil di `Application.onCreate()`
2. Check logcat untuk error messages
3. Verify Firebase Database URL correct

### Problem: Room database error
**Solution:**
1. Clean build: `./gradlew clean build`
2. Uninstall app dan install ulang
3. Check entity definitions match

## Best Practices

1. **Selalu gunakan Repository** - Jangan langsung akses DAO atau Firebase
2. **Handle exceptions** - Firebase calls dapat gagal saat offline
3. **Use coroutines** - All database operations should be in suspend functions
4. **Observe dengan Flow** - Untuk real-time UI updates
5. **Test offline mode** - Matikan internet dan test functionality

## Migration dari Firestore

Jika sebelumnya menggunakan Firestore:
1. All `FirebaseFirestore` references sudah diganti dengan `FirebaseDatabase`
2. Update security rules di Firebase Console
3. Data structure tetap sama (JSON-based)
4. Query syntax berbeda (gunakan `orderByChild()` dll)

## Resources

- [Firebase Realtime Database Docs](https://firebase.google.com/docs/database)
- [Room Database Guide](https://developer.android.com/training/data-storage/room)
- [Kotlin Coroutines & Flow](https://kotlinlang.org/docs/coroutines-guide.html)

## Support

Jika ada pertanyaan atau issue:
1. Check logcat untuk error details
2. Verify Firebase Console shows data
3. Test dengan internet ON/OFF
4. Review `DATABASE_SCHEMA.md` untuk structure details

