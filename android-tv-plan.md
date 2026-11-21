# Rencana Teknis Aplikasi Android / Android TV

Dokumen ini merangkum rencana arsitektur, pemetaan modul, tahapan implementasi, dan risiko teknis untuk membangun aplikasi Android / Android TV berbasis fitur katalog dan manajemen perpustakaan dari repositori Streambert.

## Batasan Cakupan

- **Masuk cakupan:** penjelajahan katalog dan pencarian berbasis TMDB (film & serial TV), halaman detail (sinopsis, pemeran, rating usia, trailer resmi), perpustakaan lokal (watchlist, riwayat, status sudah ditonton), kustomisasi tampilan/pengaturan, serta navigasi Android TV (D-pad / remote).
- **Di luar cakupan:** modul streaming/scraping dan pengunduhan konten berhak cipta dari sumber pihak ketiga tidak berlisensi (`src/ipc/player.js`, `src/ipc/hianime.js`, `src/ipc/downloads.js`, serta daftar sumber streaming di `src/utils/api.js`), termasuk adblock webview untuk situs-situs tersebut (`index.js`, `src/ipc/blockStats.js`, `src/ipc/siteTweaks.js`).
- **Penanganan kunci TMDB:** token TMDB tidak ditanamkan langsung (*hardcoded*) di dalam `.apk` karena mudah diekstrak dari biner dan berisiko menyalahi ketentuan distribusi kredensial. Gunakan input token di layar setup (seperti `src/components/SetupScreen.jsx`) yang disimpan di penyimpanan terenkripsi perangkat, atau lewatkan request melalui backend proxy milik sendiri.

## 1. Pilihan Stack & Arsitektur Android TV

Repositori ini dibangun menggunakan Electron (`index.js`, `preload.js`, `src/ipc/`) yang hanya menargetkan desktop (Windows, macOS, Linux), sehingga tidak dapat langsung dikemas menjadi `.apk`. Dua jalur arsitektur yang paling relevan:

1. **Opsi A — Native Android TV (Kotlin + Jetpack Compose for TV + Room + DataStore + Retrofit/Ktor) *(Direkomendasikan)***
   - Performa lebih ringan pada perangkat Android TV dengan RAM terbatas.
   - Dukungan fokus D-pad dan layout layar TV bawaan melalui `androidx.tv:tv-material`.
   - Pemutaran trailer resmi atau berkas media lokal milik pengguna melalui `androidx.media3` (`ExoPlayer`).
2. **Opsi B — Hybrid Web (Capacitor membungkus hasil `vite build`)**
   - Memanfaatkan ulang sebagian komponen React di `src/` (`HomePage.jsx`, `MoviePage.jsx`, `TVPage.jsx`, `LibraryPage.jsx`, `SettingsPage.jsx`).
   - Tetap memerlukan penggantian seluruh jembatan `window.electron` (`preload.js`) dengan plugin native Android, penyesuaian CSS untuk layar TV 10-foot, serta perluasan `src/utils/gamepadSpatialNav.js` untuk `KeyEvent` D-pad Android TV.
   - Risiko utama: konsumsi RAM WebView lebih tinggi dan konsistensi fokus D-pad lebih sulit dijaga dibanding Compose for TV.

Rencana di bawah menggunakan **Opsi A (Kotlin + Jetpack Compose for TV)**.

## 2. Pemetaan Fitur Repositori ke Modul Android

| Sumber di Repositori | Tanggung Jawab | Implementasi di Android / Android TV |
| --- | --- | --- |
| `index.js`, `src/App.jsx`, `src/components/Sidebar.jsx` | Shell aplikasi dan navigasi utama | `MainActivity` tunggal, deklarasi `LEANBACK_LAUNCHER` + `android.software.leanback` (`required="false"` bila juga mendukung ponsel/tablet) di `AndroidManifest.xml`, serta `NavigationDrawer` TV dengan rute `Home`, `Search`, `Library`, dan `Settings`. |
| `src/components/SetupScreen.jsx`, `src/ipc/storage.js` | Input dan validasi TMDB Read Access Token | Layar onboarding yang memvalidasi token ke `/3/configuration` dan `/3/trending/movie/week`, lalu menyimpannya di `EncryptedSharedPreferences` / Android Keystore. Di TV, tambahkan opsi pairing lokal (LAN/QR) agar pengguna dapat menempelkan JWT dari ponsel tanpa mengetik lewat remote. |
| `src/utils/api.js` (bagian TMDB), `src/utils/useCredits.js`, `src/utils/useRatings.js`, `src/utils/ageRating.js` | Klien metadata TMDB dan filter rating usia | Klien TMDB (`trending`, `search`, `movie/{id}`, `tv/{id}`, `credits`, `release_dates`, `content_ratings`, `videos`) memakai `OkHttp` + `Retrofit`/`Ktor`, cache HTTP disk, dan `Coil` untuk poster/backdrop. |
| `src/utils/storage.js`, `src/utils/homeLayout.js`, `src/utils/backup.js` | Penyimpanan lokal, urutan baris beranda, dan ekspor/impor | Entitas terstruktur (`SAVED`, `HISTORY`, `WATCHED`, `WATCH_PROGRESS`) di database `Room`; preferensi sederhana (`TMDB_LANG`, `AGE_LIMIT`, `RATING_COUNTRY`, `HOME_ROW_ORDER`, `THEME`, `ACCENT_COLOR`, `FONT_SIZE`) di `Preferences DataStore`; kompatibilitas format ekspor/impor JSON. |
| `src/components/TrailerModal.jsx`, pemindai berkas lokal di `src/pages/DownloadsPage.jsx` | Pemutaran trailer resmi dan media lokal milik pengguna | Buka trailer resmi YouTube melalui intent aplikasi YouTube TV / embed resmi, dan gunakan `androidx.media3` (`ExoPlayer`) jika fitur pemutar berkas video lokal dipertahankan. |
| `src/utils/gamepadSpatialNav.js`, `src/utils/useGamepadNav.js` | Navigasi arah / remote | Digantikan oleh sistem fokus bawaan Compose for TV (`FocusRequester`, `TvLazyRow` / `LazyColumn`, serta penanganan tombol `Back` dan `DPAD_CENTER`). |

## 3. Struktur Paket Proyek Android

```text
app/src/main/java/<package>/
├── data/
│   ├── tmdb/          # DTO TMDB, interceptor Authorization Bearer, repository katalog
│   └── local/         # Entity & DAO Room (watchlist, riwayat, progres), DataStore, penyimpanan token terenkripsi
├── domain/            # Filter rating usia (porting ageRating.js) & susunan baris beranda (homeLayout.js)
└── ui/
    ├── theme/         # Tema gelap dan aksen kustom (mengacu global.css & appearance.js)
    └── screens/       # SetupScreen, HomeScreen, SearchScreen, DetailScreen, LibraryScreen, SettingsScreen
```

## 4. Tahapan Implementasi & Verifikasi

1. **Inisialisasi proyek Android TV**
   - Buat proyek Gradle dengan Kotlin, Jetpack Compose, `androidx.tv:tv-material`, Navigation, Room, DataStore, Coil, dan Retrofit/Ktor.
   - Siapkan aset `banner` TV (`320x180dp`) dan konfigurasi `AndroidManifest.xml` untuk Android TV.
2. **Lapisan data & autentikasi TMDB**
   - Implementasikan penyimpanan token terenkripsi, validasi token ke TMDB, repository metadata, caching gambar/HTTP, serta logika filter rating usia.
3. **Alur UI utama di layar TV**
   - Bangun `SetupScreen` (termasuk mekanisme pairing token dari ponsel), `NavigationDrawer`, `HomeScreen` (carousel trending dan baris kategori), `SearchScreen`, dan `DetailScreen` (film & serial beserta daftar episode metadata dan trailer) dengan indikator fokus D-pad yang jelas.
4. **Perpustakaan lokal & pengaturan**
   - Hubungkan fitur simpan/watchlist, riwayat tontonan, pilihan bahasa metadata, kontrol rating usia, kustomisasi tema/aksen, dan ekspor/impor JSON.
5. **Pengujian & build `.apk`**
   - Jalankan unit test untuk parser TMDB, filter rating usia, dan DAO `Room`.
   - Uji seluruh alur di Android TV Emulator (API 33/34) hanya dengan navigasi remote (atas, bawah, kiri, kanan, OK, Back) tanpa mouse.
   - Bangun `.apk` dengan `./gradlew assembleDebug` atau `./gradlew assembleRelease` (konfigurasi keystore/signing terpisah di luar VCS) lalu verifikasi instalasi melalui `adb install`.

## 5. Risiko Teknis Utama

- **Input token di Android TV:** mengetik JWT TMDB yang panjang menggunakan remote sangat tidak praktis; pairing lokal melalui LAN atau QR code diperlukan agar onboarding nyaman dipakai di TV.
- **Kinerja perangkat TV kelas bawah:** poster beresolusi terlalu besar dan animasi berat dapat memicu *jank* atau OOM. Gunakan ukuran gambar TMDB yang proporsional (`w342`/`w500` untuk kartu, `w1280` untuk backdrop) dan batasi ukuran cache memori/disk `Coil`.
- **Perbedaan navigasi sentuh vs D-pad:** bila satu `.apk` dipakai di ponsel dan Android TV sekaligus, komponen input dan drawer perlu memisahkan interaksi sentuh dan D-pad agar tidak ada kontrol yang gagal dijangkau oleh remote.
