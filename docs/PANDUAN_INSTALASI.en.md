# Installation & Deployment Guide — IndoTech-Absensi V1

This document explains how to install **IndoTech-Absensi V1** in both a development environment and a production server.

---

## 1. System Requirements

### Required
- PHP **8.2** or newer
- Composer 2.x
- MariaDB 10.3+ / MySQL 5.7+ (default `.env.example`: `DB_CONNECTION=mariadb`)
- Node.js 18+ and NPM
- PHP extensions: `bcmath`, `ctype`, `curl`, `dom`, `fileinfo`, `gd`, `mbstring`, `openssl`, `pdo_mysql`, `tokenizer`, `xml`, `zip`

### Optional
- Redis (cache/session/queue) for production load
- Nginx or Apache web server with the document root pointing to `public/`

> `gd` is required for image processing, and `fileinfo` for file uploads.

---

## 2. Installation Steps

### 2.1 Get the source code

```bash
git clone https://github.com/Ismail-IndoTech/IndoTech-Absensi_V1.git
cd IndoTech-Absensi_V1
```

### 2.2 Backend dependencies

```bash
composer install
```

### 2.3 Environment configuration

```bash
cp .env.example .env
php artisan key:generate
```

Adjust the following values in `.env`:

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

### 2.4 Prepare the database

Create an empty database, then run:

```bash
php artisan migrate
```

To populate initial data (divisions, positions, locations, sample accounts):

```bash
php artisan migrate --seed
```

> **Warning:** `DatabaseSeeder` contains sensitive sample employee data. Do not run the seeder as-is in production. Review, replace, or delete unnecessary data, and replace all default passwords.

### 2.5 Link storage

Attendance photos are stored on the `public` disk (`storage/app/public`), so the symlink must be created:

```bash
php artisan storage:link
```

### 2.6 Build frontend assets

```bash
npm install
npm run build
```

### 2.7 Run the application

Development:

```bash
php artisan serve
```

Production: point the web server document root to the `public/` folder, then run the optimizations:

```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan queue:restart
```

### 2.8 Queue worker (optional but recommended)

Because `QUEUE_CONNECTION=database`, run the worker through Supervisor/systemd:

```bash
php artisan queue:work --tries=3 --timeout=120
```

---

## 3. Additional Configuration

### 3.1 Minimum application version

The `CheckAppVersion` middleware compares the version in the `IndoTech Browser/<version>` User-Agent with:

```php
config('app.min_app_version', '1.3.0');
```

Set the `MIN_APP_VERSION` value in `config/app.php`/`.env` if you want to raise the minimum version. Applications below the minimum version will receive a **426 Upgrade Required** response along with the APK download link.

### 3.2 APK files and landing page

- Download page: `/download-app`
- APK files are placed in `public/app-mobile/release/`
- Landing page assets are located in `public/app-mobile/`

Make sure the file names in the download links match the released APK.

### 3.3 Face detection models

The `face-api.js` models are already available in `public/models/` (`tiny_face_detector`, `face_landmark_68`, and others). The attendance page loads `face-api.js` and the models from the `/models` path. Do not delete this folder.

### 3.4 Time zone

Attendance rules depend on server time. Set `APP_TIMEZONE=Asia/Jakarta` (or the appropriate zone) so that schedule calculations, lateness, and cross-day shifts are correct.

---

## 4. Production Deployment

Brief checklist:

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

- Enable **HTTPS**; browser geolocation and camera require a secure context (`https` or `localhost`).
- Restrict public access to the `storage/` directory and `.env`.
- Make sure the `storage/` and `bootstrap/cache/` permissions are writable by the web server user.
- Back up the database regularly, especially the `attendances` and `app_logs` tables.

---

## 5. Troubleshooting

| Symptom | Common cause | Solution |
|---------|--------------|----------|
| `419 Page Expired` | Invalid session/CSRF | Check `SESSION_DRIVER`, domain, and cookies; clear the config cache. |
| Attendance photo not displayed | Storage symlink not created | Run `php artisan storage:link`. |
| Camera/location not active | Page not on HTTPS or permission denied | Use HTTPS and allow camera/location access in the browser. |
| Face detection fails | Models not loaded | Make sure `public/models` is complete and accessible. |
| Attendance rejected "outside radius" | GPS coordinates/accuracy do not match | Check the location and radius master data, as well as device accuracy. |
| Employee login rejected | Different device UUID | Use a registered device or reset the UUID through the administrator. |
| Application requests an update | Version below the minimum | Update the APK, or adjust `min_app_version`. |
| Data does not appear after seeding | Seeder not run | Run `php artisan db:seed`. |

---

## 6. Testing

```bash
php artisan test
```

The built-in tests are still samples (`tests/Feature`, `tests/Unit`). Add tests for the attendance and request flows before making major changes.
