# Panduan Instalasi & Deployment — IndoTech-Absensi V1

Dokumen ini menjelaskan cara memasang **IndoTech-Absensi V1** pada lingkungan pengembangan maupun server produksi.

---

## 1. Persyaratan Sistem

### Wajib
- PHP **8.2** atau lebih baru
- Composer 2.x
- MariaDB 10.3+ / MySQL 5.7+ (default `.env.example`: `DB_CONNECTION=mariadb`)
- Node.js 18+ dan NPM
- Ekstensi PHP: `bcmath`, `ctype`, `curl`, `dom`, `fileinfo`, `gd`, `mbstring`, `openssl`, `pdo_mysql`, `tokenizer`, `xml`, `zip`

### Opsional
- Redis (cache/session/queue) untuk beban produksi
- Web server Nginx atau Apache dengan document root ke `public/`

> `gd` diperlukan untuk pemrosesan gambar, dan `fileinfo` untuk unggahan berkas.

---

## 2. Langkah Instalasi

### 2.1 Ambil kode sumber

```bash
git clone https://github.com/Ismail-IndoTech/IndoTech-Absensi_V1.git
cd IndoTech-Absensi_V1
```

### 2.2 Dependensi backend

```bash
composer install
```

### 2.3 Konfigurasi environment

```bash
cp .env.example .env
php artisan key:generate
```

Sesuaikan nilai berikut pada `.env`:

```dotenv
APP_NAME="IndoTech Absensi"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://absensi.contoh.com
APP_TIMEZONE=Asia/Jakarta

DB_CONNECTION=mariadb
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=absensi_gsg
DB_USERNAME=absensi
DB_PASSWORD=GANTI_DENGAN_PASSWORD_KUAT

SESSION_DRIVER=database
CACHE_STORE=database
QUEUE_CONNECTION=database
```

### 2.4 Siapkan database

Buat database kosong, lalu jalankan:

```bash
php artisan migrate
```

Untuk mengisi data awal (divisi, jabatan, lokasi, akun contoh):

```bash
php artisan migrate --seed
```

> **Peringatan:** `DatabaseSeeder` berisi data contoh karyawan yang bersifat sensitif. Jangan menjalankan seeder apa adanya di produksi. Tinjau, ganti, atau hapus data yang tidak diperlukan, dan ganti semua kata sandi bawaan.

### 2.5 Tautkan storage

Foto presensi disimpan pada disk `public` (`storage/app/public`), sehingga symlink wajib dibuat:

```bash
php artisan storage:link
```

### 2.6 Build aset frontend

```bash
npm install
npm run build
```

### 2.7 Jalankan aplikasi

Pengembangan:

```bash
php artisan serve
```

Produksi: arahkan document root web server ke folder `public/`, lalu jalankan optimasi:

```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan queue:restart
```

### 2.8 Worker antrean (opsional namun disarankan)

Karena `QUEUE_CONNECTION=database`, jalankan worker melalui Supervisor/systemd:

```bash
php artisan queue:work --tries=3 --timeout=120
```

---

## 3. Konfigurasi Tambahan

### 3.1 Versi minimum aplikasi

Middleware `CheckAppVersion` membandingkan versi pada User-Agent `IndoTech Browser/<versi>` dengan:

```php
config('app.min_app_version', '1.3.0');
```

Atur nilai `MIN_APP_VERSION` pada `config/app.php`/`.env` bila ingin menaikkan versi minimum. Aplikasi di bawah versi minimum akan menerima respons **426 Upgrade Required** beserta tautan unduhan APK.

### 3.2 Berkas APK dan landing page

- Halaman unduhan: `/download-app`
- Berkas APK ditempatkan pada `public/app-mobile/release/`
- Aset landing page berada di `public/app-mobile/`

Pastikan nama berkas pada tautan unduhan sesuai dengan APK yang dirilis.

### 3.3 Model deteksi wajah

Model `face-api.js` sudah tersedia di `public/models/` (`tiny_face_detector`, `face_landmark_68`, dan lainnya). Halaman presensi memuat `face-api.js` dan model dari path `/models`. Jangan menghapus folder ini.

### 3.4 Zona waktu

Aturan presensi bergantung pada waktu server. Setel `APP_TIMEZONE=Asia/Jakarta` (atau zona yang sesuai) agar perhitungan jadwal, keterlambatan, dan shift lintas hari benar.

---

## 4. Deployment Produksi

Checklist ringkas:

```bash
composer install --no-dev --optimize-autoloader
npm ci && npm run build
php artisan migrate --force
php artisan storage:link
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan queue:restart
```

- Aktifkan **HTTPS**; geolocation dan kamera browser mensyaratkan konteks aman (`https` atau `localhost`).
- Batasi akses direktori `storage/` dan `.env` dari publik.
- Pastikan permission `storage/` dan `bootstrap/cache/` dapat ditulis oleh user web server.
- Cadangkan database secara berkala, terutama tabel `attendances` dan `app_logs`.

---

## 5. Pemecahan Masalah

| Gejala | Penyebab umum | Solusi |
|--------|----------------|--------|
| `419 Page Expired` | Session/CSRF tidak valid | Periksa `SESSION_DRIVER`, domain, dan cookie; bersihkan cache config. |
| Foto presensi tidak tampil | Symlink storage belum dibuat | Jalankan `php artisan storage:link`. |
| Kamera/lokasi tidak aktif | Halaman tidak di HTTPS atau izin ditolak | Gunakan HTTPS dan izinkan akses kamera/lokasi di browser. |
| Deteksi wajah gagal | Model tidak termuat | Pastikan `public/models` lengkap dan dapat diakses. |
| Absen ditolak "di luar radius" | Koordinat/akurasi GPS tidak sesuai | Periksa master lokasi dan radius, serta akurasi perangkat. |
| Login karyawan ditolak | UUID perangkat berbeda | Gunakan perangkat terdaftar atau reset UUID melalui administrator. |
| Aplikasi meminta update | Versi di bawah minimum | Perbarui APK, atau sesuaikan `min_app_version`. |
| Data tidak muncul setelah seeding | Seeder tidak dijalankan | Jalankan `php artisan db:seed`. |

---

## 6. Pengujian

```bash
php artisan test
```

Test bawaan masih berupa contoh (`tests/Feature`, `tests/Unit`). Tambahkan pengujian untuk alur presensi dan pengajuan sebelum melakukan perubahan besar.
