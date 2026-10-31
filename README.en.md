> 📚 **Documentation:** [Full ID/EN index](DOCS.md) · [English index](DOCS.en.md)

<div align="center">

# IndoTech-Absensi V1


### Web-Based and APK Employee Attendance Application — Geofencing, Face Verification, Schedules & Shifts, Leave Requests, and Administrator Reports.

[![License: MIT](https://img.shields.io/badge/license-MIT-0EA5E9?style=flat-square)](LICENSE)
[![PHP](https://img.shields.io/badge/PHP-8.2%2B-777BB4?style=flat-square&logo=php)](#)
[![Laravel](https://img.shields.io/badge/Laravel-12.x-FF2D20?style=flat-square&logo=laravel)](#)
[![Livewire](https://img.shields.io/badge/Livewire-3.x-FB70A9?style=flat-square&logo=livewire)](#)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind-3.x-38B2AC?style=flat-square&logo=tailwind-css)](#)
[![Made in Indonesia](https://img.shields.io/badge/made%20in%20Indonesia-FF2D20?style=flat-square)](https://github.com/Ismail-IndoTech)

**The first version of the IndoTech attendance application line: daily attendance with location validation, client-side face detection, work scheduling, leave/overtime/shift-swap requests, and an integrated administrator panel.**

</div>

---

## 📖 Overview

**IndoTech-Absensi V1** is an employee attendance application built on **Laravel 12 + Livewire 3** that provides two interfaces within a single codebase:

1. **Administrator Panel** — manages master data (employees, divisions, positions, attendance locations, radius, schedules, shifts, holidays), reviews attendance reports, and processes employee requests.
2. **Employee Panel** — performs clock-in/clock-out, views history, submits leave/overtime/schedule-change requests, and monitors work schedules.

The application also provides an **APK download landing page** (`/download-app`) as well as an **API** module for mobile application needs, with device binding and an activity audit log.

---

## ✨ Key Features

### Attendance & Validation
- **Clock-in** and **clock-out** with schedule validation (no check-in more than 45 minutes before the start time, check-out at least 30 minutes after check-in, and a maximum of 8 hours after the schedule ends).
- **Face detection** in the browser using `face-api.js` (models available locally in `public/models`).
- **Location validation** with **Leaflet + OpenStreetMap** maps, office radius geofencing, and GPS accuracy checks.
- Support for **cross-day shifts** (`next_day`), including determining the active schedule based on the previous day's attendance.
- Detection of **mock location** indications on the attendance page.

### Scheduling
- Master **fixed schedules** and **shifts** along with schedule patterns (`schedule_patterns`, `shift_patterns`, `shift_groups`).
- **Schedule change requests** and **shift swaps** by employees, with an administrator approval flow.
- Master **holidays** that are taken into account in attendance calculations.

### Employee Requests
- **Leave/sick/time-off** requests, **overtime**, and **overtime outside the schedule**.
- Request status and request history on the employee panel.
- Approval/rejection of requests through the administrator panel.

### Administration & Reporting
- Attendance reports and request reports.
- **Employee data import** from spreadsheet files (`App\Imports` module).
- Master data for employees, divisions, positions, sub-positions, attendance locations, and radius.
- **Activity audit log** (`app_logs`) for login and attendance activities.
- **Notifications** for employees (web and API).

### Security
- Two separate authentication guards: `admin` (`users` table) and `employee` (`employees` table).
- **Device binding** using a device UUID; employees cannot log in from another device.
- **User-Agent** restriction on Android: employees are directed to use the application rather than a regular browser.
- **Login attempt limiting**: 3 failures per email (60-second block) and 5 different emails per IP (15-minute block).
- **Application version check** via the `CheckAppVersion` middleware (the minimum version can be set through `config('app.min_app_version')`).
- **Root detection** in the login flow (`rootDetected`) to direct the user to special handling.

---

## 👥 User Roles

| Role | Guard | Access |
|------|-------|--------|
| Administrator | `admin` | `/admin/*` panel: attendance reports, permissions, master data, employee import, user management. |
| Employee | `employee` | Dashboard, attendance, history, schedule, requests, profile. |

---

## 🛠️ Technology

- **Backend**: PHP 8.2+, Laravel 12, Livewire 3, Laravel Sanctum
- **Frontend**: Blade, Tailwind CSS 3, Vite 6, Alpine.js (bundled with Livewire)
- **Attendance**: `face-api.js` (face detection), Leaflet + OpenStreetMap, Geolocation API
- **Database**: MariaDB/MySQL (default `mariadb` in `.env.example`)
- **Additional**: `jenssegers/agent` (device detection), PhpSpreadsheet (Excel import), PHPUnit 11

---

## ✅ Requirements

- PHP **8.2+** with the `bcmath`, `ctype`, `curl`, `dom`, `fileinfo`, `gd`, `mbstring`, `openssl`, `pdo_mysql`, `tokenizer`, `xml`, `zip` extensions
- Composer 2.x
- Node.js 18+ and NPM
- MariaDB/MySQL
- Web server (Apache/Nginx) or `php artisan serve` for development

---

## 🚀 Quick Installation

```bash
# 1. Clone the repository (private)

git clone https://github.com/Ismail-IndoTech/IndoTech-Absensi_V1.git
cd IndoTech-Absensi_V1

# 2. Backend dependencies

composer install

# 3. Environment configuration

cp .env.example .env
php artisan key:generate

# 4. Adjust the database connection in .env

#    DB_CONNECTION=mariadb

#    DB_DATABASE=absensi_gsg

#    DB_USERNAME=...  DB_PASSWORD=...


# 5. Migrate and seed initial data

php artisan migrate --seed

# 6. Link storage for attendance photos

php artisan storage:link

# 7. Frontend dependencies and asset build

npm install
npm run build

# 8. Run

php artisan serve
```

Open `http://127.0.0.1:8000`. The APK download page is available at `/download-app`.

> **Important:** `DatabaseSeeder` loads sample administrator and employee data. Review and adjust its contents before running it in a production environment, then replace all default credentials.

A step-by-step guide is available in **[docs/PANDUAN_INSTALASI.en.md](docs/PANDUAN_INSTALASI.en.md)**.

---

## ▶️ Running During Development

```bash
# Server, queue listener, and Vite at once

composer run dev
```

Or separately:

```bash
php artisan serve
npm run dev
php artisan queue:work
```

The default `QUEUE_CONNECTION` is `database`, so a worker is required if queue features are used.

---

## 📂 Project Structure

```text
IndoTech-Absensi_V1/
├── app/
│   ├── Http/
│   │   ├── Controllers/          # Auth, Admin, User, Api
│   │   └── Middleware/           # CheckAppVersion, LoginAttempt, CheckDeviceType
│   ├── Imports/                  # Employee data import (spreadsheet)
│   ├── Livewire/
│   │   ├── Admin/                # Dashboard, master data, reports
│   │   └── User/                 # Attendance, history, schedule, requests, profile
│   ├── Models/                   # Employee, Attendance, Schedule, Shift, etc.
│   └── Traits/
├── config/                       # Application & authentication configuration
├── database/
│   ├── migrations/               # Attendance, schedule, shift, request schema
│   └── seeders/                  # Initial data
├── public/
│   ├── models/                   # face-api.js models
│   ├── app-mobile/               # Landing page & APK files
│   ├── admin/ · user/ · auth/    # Interface assets
│   └── index.php
├── resources/
│   ├── css/ · js/
│   └── views/                    # Blade: admin, user, auth, components
├── routes/
│   ├── web.php                   # Admin & employee routes
│   ├── api.php                   # API routes (notifications, token)
│   └── console.php
├── storage/
└── tests/
```

---

## 🔄 Brief Attendance Flow

1. The employee logs in (guard `employee`) with a bound device UUID.
2. The attendance page loads the camera, map, office radius, and face detection models.
3. The attendance button becomes active once the camera is ready, the map is ready, a face is detected, and the location is valid.
4. Data is sent to `POST /absensi` (or `POST /absensiOvTm` for overtime), stored as an `Attendance` along with the photo (base64 → `storage/app/public`) and coordinates.
5. The system determines the clock-in/clock-out status based on the active schedule or shift, including cross-day shifts.
6. Every important activity is recorded in `app_logs`.

Detailed rules and modules are explained in **[docs/PANDUAN_FITUR.en.md](docs/PANDUAN_FITUR.en.md)**.

---

## 🔌 API & Notifications

- API tokens are generated through `App\Http\Controllers\Api\AuthController` (Laravel Sanctum).
- Notification endpoints: `GET /api/notifications`, `POST /api/notifications/{id}/read`, `POST /api/notifications/{id}/respond`.
- API routes use the `auth:api` and `check.app.version` middleware; make sure those guards/middleware are configured according to your deployment needs before using them in production.

---

## 🔐 Security Notes

- Do not commit `.env`, credentials, or employees' personal data.
- Review `database/seeders/DatabaseSeeder.php` — this file contains sensitive sample data and is **not for production as-is**.
- Replace all default passwords and enable HTTPS on deployment.
- Client-side face detection and location validation are additional layers, not a replacement for server-side validation.

---

## 📚 Documentation

- 📖 **[Installation & Deployment Guide](docs/PANDUAN_INSTALASI.en.md)**
- 📖 **[Feature & Workflow Guide](docs/PANDUAN_FITUR.en.md)**

---

## 💖 Contribution & Donation

If this application makes IT operations, DevOps, and server management easier for your company or personal projects, support its development by becoming a contributor or a **Special Donor**.
*(Official donors and sponsors will have their company logos listed in this repository).*

Support ongoing maintenance, research into new panel bypass modules, and system updates by donating through:

<div align="center">

**DANA — 08121600058** (a.n. Mochamad Ismail Soleh)

<img src="docs/qris.jpg" alt="IndoTech DANA QRIS" width="280">

*Scan with the DANA, BCA, Mandiri, GoPay, OVO, or any other QRIS payment application in Indonesia.*

</div>

All funds are allocated to server research infrastructure costs, testing tool licenses, and the development of IndoTech open-source modules. If you or your institution would like to become an official partner or sponsor, please contact us. Thank you for your support!

---

## 📄 License

This project is released under the open [MIT License](LICENSE).
Copyright © 2026 **Mochamad Ismail Soleh (Ismail.IndoTech)**.

<div align="center">

Made with ❤️ by [**Ismail.IndoTech**](https://it-app.id) · [it-app.id](https://it-app.id)

</div>