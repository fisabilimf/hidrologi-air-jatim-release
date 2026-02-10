# Hidrologi Mobile App

Aplikasi mobile **Hidrologi Air** berbasis **Flutter**.

Repo ini mencakup:

- Aplikasi Flutter (Android/iOS + target desktop/web tersedia dari struktur proyek)
- Integrasi **Firebase Cloud Messaging (FCM)** untuk push notifications berbasis **topic**
- Peta menggunakan **Google Maps**

> Catatan kompatibilitas SDK
>
> Proyek ini masih memakai constraint Dart **`>=2.7.0 <3.0.0`** (lihat `pubspec.yaml`).
> Artinya gunakan Flutter yang masih membawa **Dart 2.x**. Jika kamu memakai Flutter terbaru (Dart 3), kamu perlu migrasi (update constraint + perbaikan kode/dependencies).

## Prasyarat

- Flutter SDK (Dart 2.x sesuai constraint di `pubspec.yaml`)
- Android Studio (untuk Android SDK/Platform Tools) atau minimal Android SDK + command line tools
- (Opsional) Xcode untuk iOS (hanya di macOS)
- Git

## Instalasi (setup project)

1) Clone repo

1) Ambil dependencies

- Jalankan: `flutter pub get`

1) Pastikan device/emulator siap

- Android: buka Android Emulator atau colok device dan pastikan terdeteksi (`flutter devices`)

## Menjalankan aplikasi (dev)

### Android (tanpa flavor)

Jika kamu **tidak** memakai product flavor, kamu bisa jalankan:

- `flutter run`

Namun, project ini sudah mendefinisikan **product flavors** untuk konfigurasi Firebase Android (lihat bagian Firebase/FCM). Umumnya kamu akan menjalankan salah satu flavor di bawah.

### Android (dengan flavor)

Di `android/app/build.gradle` ada 2 flavor:

- `deviceStreaming`
- `monitoringStatus`

Contoh menjalankan:

- `flutter run --flavor deviceStreaming`
- `flutter run --flavor monitoringStatus`

Jika kamu punya beberapa `main_*.dart`, biasanya perlu menambahkan `-t lib/main_xxx.dart`. Di repo ini entrypoint default adalah `lib/main.dart`.

### iOS

Di macOS:

- `flutter run -d ios`

### Web (run)

- `flutter run -d chrome`

### Windows (run)

- `flutter run -d windows`

## Build (rilis)

### Android APK

- Tanpa flavor: `flutter build apk --release`
- Dengan flavor:
  - `flutter build apk --release --flavor deviceStreaming`
  - `flutter build apk --release --flavor monitoringStatus`

Output umumnya ada di `build/app/outputs/flutter-apk/`.

### Android App Bundle (Play Store)

- Tanpa flavor: `flutter build appbundle --release`
- Dengan flavor:
  - `flutter build appbundle --release --flavor deviceStreaming`
  - `flutter build appbundle --release --flavor monitoringStatus`

Output umumnya ada di `build/app/outputs/bundle/release/`.

> Signing release
>
> Saat ini `buildTypes.release` masih memakai debug signing (lihat `android/app/build.gradle`).
> Untuk rilis produksi, buat keystore sendiri dan konfigurasi signing release.

## Optimasi ukuran & performa (HP lama)

Di perangkat lama, dua hal yang paling terasa:

1) **Ukuran download / ukuran instalasi** (APK/AAB)
2) **Performa runtime** (startup lambat, scroll patah-patah, gampang OOM)

### 1) Kecilkan ukuran APK/AAB Android

Repo ini sudah mengaktifkan **R8/ProGuard** + **resource shrinking** untuk build `release` (lihat `android/app/build.gradle`).
Langkah berikutnya yang biasanya paling berdampak:

- **Gunakan split per ABI saat build APK** (hasilnya beberapa APK, masing-masing lebih kecil):
  - `flutter build apk --release --flavor deviceStreaming --split-per-abi`
  - `flutter build apk --release --flavor monitoringStatus --split-per-abi`

- **Untuk Play Store, prefer App Bundle (AAB)** karena Play akan melakukan split otomatis (ABI/density/language):
  - `flutter build appbundle --release --flavor deviceStreaming`
  - `flutter build appbundle --release --flavor monitoringStatus`

- **Strip simbol debug Dart untuk memperkecil lib AOT** (recommended untuk rilis). Ini tidak mengubah fungsi aplikasi, tapi membuat stacktrace butuh symbol file:
  - `flutter build apk --release --flavor deviceStreaming --split-per-abi --split-debug-info=build/symbols/deviceStreaming`
  - `flutter build appbundle --release --flavor deviceStreaming --split-debug-info=build/symbols/deviceStreaming`

  (Opsional) tambah `--obfuscate` jika kamu siap mengelola symbol/mapping untuk crash reports.

- **Analisis apa yang bikin bengkak**:
  - `flutter build apk --release --flavor deviceStreaming --analyze-size`
  Lalu buka DevTools (tab *App Size*) untuk melihat komponen mana yang paling besar.

### 2) Optimasi asset (gambar)

Gambar besar sering bikin aplikasi terasa berat karena decoding + memori.
Praktik yang biasanya paling membantu:

- Kompres PNG/JPG (tanpa mengubah dimensi kalau tidak perlu).
- Pastikan kamu tidak menampilkan gambar jauh lebih besar dari ukuran widget (hindari decode 4K untuk ditampilkan 200px).
- Pertimbangkan **WebP** untuk Android jika kualitas masih oke.

### 3) Optimasi performa runtime (yang paling terasa di HP lama)

Beberapa kebiasaan di Flutter yang dampaknya besar:

- Gunakan `const` sebanyak mungkin pada widget statis agar rebuild lebih murah.
- Untuk list panjang, gunakan `ListView.builder`/`SliverList` (hindari membuat semua item sekaligus).
- Kurangi rebuild yang tidak perlu:
  - pecah widget besar jadi widget kecil
  - cache hasil `Future`/`Stream` (hindari membuat ulang request pada setiap build)
- Untuk gambar jaringan: sudah memakai `cached_network_image`; pastikan ukuran cache wajar dan hindari memuat gambar resolusi besar.
- Hindari animasi berat / blur / shadow berlapis pada layar yang sering dibuka.

### 4) Catatan dependencies

Beberapa dependency di project ini tergolong berat (misalnya: Google Maps, WebView, PDF viewer, dan paket chart/datagrid). Kalau ada fitur yang jarang dipakai di lapangan, pertimbangkan:

- Menyederhanakan tampilan (mis. chart ringan atau ringkasan angka)
- Meminimalkan penggunaan WebView
- Memecah fitur jarang dipakai menjadi screen terpisah yang di-load saat dibuka (agar startup lebih cepat)


### iOS (IPA)

Di macOS:

- `flutter build ipa --release`

### Web (build)

- `flutter build web --release`

### Windows (build)

- `flutter build windows --release`

## Konfigurasi penting

### Firebase / FCM (Push Notifications)

Client sudah punya implementasi push notification berbasis **FCM Topics**.
Logic utamanya ada di:

- `lib/utils/push_notifications/push_notification_service.dart`

Dipanggil saat:

- login sukses (`lib/ui/login/pages/login_view.dart`)
- auto-login / startup (`lib/main.dart`)
- logout (unsubscribe) (`lib/utils/appDrawer.dart`)

### 1) Tambahkan file config Firebase (wajib)

FCM **tidak akan jalan** sampai file ini tersedia:

- **Android** (pilih salah satu pendekatan):
  - Single config: `android/app/google-services.json`, atau
  - Per-flavor config (direkomendasikan untuk project ini):
    - `android/app/src/deviceStreaming/google-services.json`
    - `android/app/src/monitoringStatus/google-services.json`
- **iOS**: `ios/Runner/GoogleService-Info.plist`

Get them from Firebase Console:

1. Create/select a Firebase project
2. Add Android app / iOS app
3. Download the config file and place it in the path above

Penting:

- Android package name di Firebase harus match `applicationId` di `android/app/build.gradle`.
  Saat ini: `com.dinotech.hidrologi`.

#### Android: multiple Firebase configs (flavors)

Karena `google-services.json` hanya mewakili **satu** Firebase project per build variant, repo ini memakai **product flavors**.
Saat menjalankan/build, pilih flavor yang sesuai (lihat contoh pada bagian *Menjalankan* dan *Build*).

### 2) Pola topic yang dipakai aplikasi

The app subscribes to topics based on the logged-in user profile (`GET /api/user`) and (for petugas lapangan) the assigned pos list (`GET /api/get_pos_saya/0/`).

Current topic patterns:

#### Base identity topics (subscribed by *all* users)

In addition to feature topics below, the app always subscribes to these stable topics (when available), so the backend can target notifications **per user / per role / per UPT** reliably:

- `user_{userId}` (if `id` exists in `GET /api/user`)
- `model_{model}` (role model from profile)
- `upt_{uptId}` (if `upt_id != 0`)
- role aliases:
  - `role_admin_pusat`
  - `role_admin_upt`
  - `role_petugas`

- Admin pusat (`model=0` and `upt_id` is null/0)
  - `all_pengajuan`
  - `all_pos_hujan`
  - `all_pos_duga`
  - (optional if you enable) `all_pos_kualitas`
  - (optional if you enable) `all_kondisi`
- Admin UPT (`model=0/1` and `upt_id != 0`)
  - `upt_{uptId}_pos_hujan`
  - `upt_{uptId}_pos_duga`
- Petugas lapangan (`model=2`)
  - per-pos topics, based on station type:
    - `pos_hujan_{posId}`
    - `pos_duga_{posId}`
    - `pos_kualitas_{posId}`

### 3) Mengirim notifikasi (server/backend)

The server should publish to the appropriate FCM topics above (recommended). This repository does not include the backend sender.

Recommended targeting approach:

- User-specific reminder (only one person): send to `user_{userId}`
- Role-based broadcast: send to `model_{model}` or `role_*`
- UPT broadcast: send to `upt_{uptId}` or `upt_{uptId}_pos_*` depending on the feature

### 4) Jadwal reminder (07:00 / 12:00 / 17:00)

FCM itself does not “schedule” messages on the device reliably (especially with Doze/background limits). The recommended approach is:

1. Use a backend cron/scheduler to send pushes at the desired time.
2. Target topics that match the intended audience.

#### Petugas pos hujan reminders

The app subscribes petugas who have **at least one pos hujan assignment** to:

- `role_petugas_pos_hujan`
- (optional, if `upt_id` exists) `upt_{uptId}_role_petugas_pos_hujan`

So you can schedule:

- 07:00 daily → send to `role_petugas_pos_hujan`
- 12:00 daily → send to `role_petugas_pos_hujan`
- 17:00 daily → send to `role_petugas_pos_hujan`

If you want UPT-specific reminders, send to `upt_{uptId}_role_petugas_pos_hujan`.

Payload guidance:

- Prefer including `notification` (for background display) and `data` (for routing).
- Ensure Android uses channel id `hidrologi_general`.

#### Admin pusat: pengajuan masuk

Admin pusat subscribes to:

- `all_pengajuan`
- `role_admin_pusat`

When a new pengajuan is created, send to `all_pengajuan` (or `role_admin_pusat` if you prefer role naming).

## Google Maps API key

Aplikasi memakai `google_maps_flutter`. Di Android, API key saat ini ditaruh sebagai `<meta-data android:name="com.google.android.geo.API_KEY" .../>` di `android/app/src/main/AndroidManifest.xml`.

Rekomendasi untuk environment baru:

- Gunakan API key milik kamu sendiri dari Google Cloud Console.
- Restrict key (Android apps restriction + SHA-1/SHA-256) agar tidak mudah disalahgunakan.

## Troubleshooting singkat

- **Build gagal karena google-services.json tidak ketemu**
  - Pastikan file ada untuk flavor yang kamu build.
  - Contoh: build `--flavor deviceStreaming` butuh `android/app/src/deviceStreaming/google-services.json`.

- **Notifikasi tidak masuk di Android 13+**
  - Pastikan permission notifikasi diberikan. Manifest sudah mencantumkan `POST_NOTIFICATIONS`, tapi runtime permission tetap perlu di-approve pengguna.

## Referensi

- Flutter docs: <https://docs.flutter.dev/>
