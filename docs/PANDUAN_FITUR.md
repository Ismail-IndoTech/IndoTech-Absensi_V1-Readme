# Panduan Fitur & Alur Kerja — IndoTech-Absensi V1

Dokumen ini menjelaskan modul, aturan bisnis, dan rute utama aplikasi.

---

## 1. Peran & Autentikasi

Aplikasi memakai dua guard terpisah pada `config/auth.php`:

| Guard | Provider | Model | Tabel | Penggunaan |
|-------|----------|-------|-------|------------|
| `admin` | `users` | `App\Models\User` | `users` | Panel administrator |
| `employee` | `employees` | `App\Models\Employee` | `employees` | Panel karyawan |

- Login tunggal di `/login`; sistem mendeteksi apakah email milik administrator atau karyawan.
- Sesi administrator berlaku 60 menit dan berakhir saat browser ditutup.
- Sesi karyawan berlaku panjang (1 tahun) agar nyaman dipakai pada perangkat absensi.
- Akun dengan `status != 0` dianggap tidak aktif dan ditolak saat login.
- Karyawan yang memakai browser Android tanpa penanda `IndoTech-APP`/`IndoTech-Browser` akan diarahkan untuk mengunduh aplikasi.

---

## 2. Panel Administrator

Rute berada pada prefix `/admin` dengan middleware `auth:admin`.

### 2.1 Laporan
- `/admin/reportPresensi` — laporan presensi karyawan.
- `/admin/perizinan` — daftar pengajuan karyawan.
- `/admin/permissions` — pemrosesan/persetujuan pengajuan.

### 2.2 Master Data
| Modul | Rute utama | Keterangan |
|-------|------------|------------|
| Lokasi absensi | `/admin/lokasi` | Titik kantor, latitude/longitude, radius, akurasi. |
| Radius | melalui lokasi | Radius geofence per lokasi. |
| Jadwal tetap | `/admin/schedule/list` | Jadwal kerja beserta pola (`schedule_patterns`). |
| Shift | `/admin/shift/list` | Shift kerja, pola shift, dan grup shift. |
| Divisi | `/admin/devisi` | Data divisi dan detail anggota. |
| Jabatan | `/admin/positions` | Daftar jabatan dan sub-jabatan. |
| Karyawan | `/admin/employee` | Tambah, ubah, nonaktifkan, dan hapus karyawan. |
| Hari libur | `/admin/holiday` | Kalender hari libur. |
| Pengguna admin | `/admin/user` | Pengelolaan akun administrator. |

### 2.3 Karyawan
- **Tambah karyawan** melalui `/admin/employee/new`.
- **Impor massal** dari spreadsheet menggunakan komponen `Admin\Dashboard\ImportDataExel` dan kelas di `app/Imports`.
- **Reset ikatan perangkat** melalui `PUT /admin/employee/rebind-employee/{id}`.
- **Ubah status** aktif/nonaktif melalui `PUT /admin/employee/{id}/update-status`.

---

## 3. Panel Karyawan

Rute dengan middleware `auth:employee`.

| Halaman | Rute | Fungsi |
|---------|------|--------|
| Dashboard | `/dashboard` | Ringkasan presensi dan jadwal hari ini. |
| Presensi | `/presensi` | Absen masuk/pulang dengan kamera, peta, dan deteksi wajah. |
| Presensi lembur | `/presensiOvTm` | Absen untuk kegiatan lembur. |
| Riwayat | `/history` | Riwayat presensi karyawan. |
| Jadwal | `/schedule` | Jadwal kerja yang berlaku. |
| Pengajuan | `/requested` | Daftar pengajuan dan statusnya. |
| Tambah pengajuan | `/requested/add` | Formulir izin, lembur, atau perubahan jadwal. |
| Profil | `/profile` | Data profil karyawan. |

---

## 4. Aturan Presensi

Logika utama berada pada `App\Http\Controllers\UserController::absensi()`.

### 4.1 Penentuan tipe absen
- Jika belum ada `attd_in_time` hari ini → **absen masuk**.
- Jika sudah ada `attd_in_time` → **absen pulang**.

### 4.2 Validasi waktu
| Aturan | Nilai | Respons |
|--------|-------|---------|
| Check-in terlalu awal | Lebih dari 45 menit sebelum jam masuk | Ditolak (kode `10`) |
| Check-out terlalu cepat | Kurang dari 30 menit setelah check-in (atau sebelum jam selesai pada shift tertentu) | Ditolak (kode `11`) |
| Melewati batas absen | Lebih dari 8 jam setelah jadwal berakhir | Ditolak (kode `12`) |

### 4.3 Shift lintas hari
- Jika pola jadwal memiliki `next_day`, jam selesai dihitung pada hari berikutnya.
- Sistem juga memeriksa jadwal kemarin: bila waktu sekarang masih berada dalam rentang shift kemarin dan belum check-out, absensi mengikuti jadwal tersebut.
- Toleransi tambahan 8 jam diberikan hanya bila karyawan belum melakukan check-out.

### 4.4 Penyimpanan data presensi
- Foto dikirim sebagai base64, didekode, lalu disimpan ke disk `public` dengan nama `dd-mm-yyyy-<tipe>.jpg`.
- Data tersimpan pada tabel `attendances`: `attd_in_time`, `attd_out_time`, `location_in`, `location_out`, `pict_in`, `pict_out`, `status_in`, `status_out`, `type`, dan relasi jadwal/shift/pengajuan.

---

## 5. Validasi Lokasi & Wajah

### 5.1 Geofence
- Peta memakai **Leaflet** dengan tile **OpenStreetMap**.
- Lokasi kantor dan radius diambil dari master `attend_locations` dan `radii`.
- Posisi pengguna dipantau dengan `navigator.geolocation.watchPosition`.
- Tombol absen nonaktif bila pengguna berada di luar radius atau akurasi GPS tidak memadai.

### 5.2 Deteksi wajah
- Menggunakan `face-api.js` dengan model `tiny_face_detector` dari `public/models`.
- Wajah harus terdeteksi dan berada dalam ukuran tertentu sebelum tombol absen aktif.
- Deteksi berjalan di browser; validasi akhir tetap dilakukan di server.

### 5.3 Indikasi mock location
- Halaman presensi memeriksa parameter `mockLocation`; bila terindikasi, tombol absen tetap nonaktif.

---

## 6. Penjadwalan

### 6.1 Jadwal tetap
- Tabel `schedules` dan `schedule_patterns`.
- Pola menentukan hari kerja, jam mulai/selesai, dan penanda `next_day`.

### 6.2 Shift
- Tabel `shifts`, `shift_patterns`, dan `shift_groups`.
- Digunakan untuk pola kerja bergilir.

### 6.3 Pengajuan perubahan
- **Perubahan jadwal** disimpan pada `schedule_requests`.
- **Tukar shift** disimpan pada `shift_requests`.
- Keduanya memiliki status persetujuan dan dapat memengaruhi presensi (`sched_req_id`, `shift_req_id`).

---

## 7. Pengajuan Karyawan

Tipe pengajuan yang dikenal sistem antara lain:

- `sick` — sakit
- `leave` — cuti/izin
- `overtime` — lembur terjadwal
- `overtimeNotSched` — lembur di luar jadwal
- `otgo` — lembur khusus

Pengajuan yang disetujui dapat menghasilkan entri presensi bertipe `requested`, `changeSched`, `overtime`, atau `overtimeNotSched`.

---

## 8. Hari Libur

- Data pada tabel `holidays`.
- Digunakan untuk menandai hari libur pada perhitungan presensi dan tampilan jadwal.

---

## 9. Notifikasi

- Tabel `notifications` menyimpan `user_id`, `type`, `message`, `data` (JSON), dan `read`.
- Tersedia endpoint API:
  - `GET /api/notifications`
  - `POST /api/notifications/{id}/read`
  - `POST /api/notifications/{id}/respond`

---

## 10. Audit Log

- Model `AppLog` mencatat aktivitas penting ke tabel `app_logs`.
- Contoh kejadian: login gagal, login berhasil dengan UUID baru, penolakan perangkat, dan penolakan presensi karena aturan waktu.
- Gunakan log ini untuk investigasi dan audit operasional.

---

## 11. Keamanan Perangkat

- **UUID perangkat**: dibuat di klien dan disimpan pada `localStorage`; nilai pertama disimpan ke kolom `uuid` karyawan. Login dari UUID berbeda ditolak.
- **Pemeriksaan versi**: middleware `CheckAppVersion` menolak aplikasi di bawah versi minimum (HTTP 426).
- **Pembatasan login**: 3 kegagalan per email → blokir 60 detik; 5 email berbeda dari satu IP → blokir 15 menit.
- **Deteksi root**: parameter `rootDetected=true` pada login diarahkan ke penanganan khusus.
- **Pemeriksaan sesi**: `AppServiceProvider` memeriksa `session_expiration` dan mengeluarkan pengguna bila sesi kedaluwarsa.

---

## 12. Catatan Pengembangan

- Folder `tests/` masih berisi contoh pengujian; tambahkan feature test untuk alur kritis (login, presensi, pengajuan).
- Beberapa berkas memiliki salinan bertanda `(1)` (mis. `Dashboard(1).php`). Berkas tersebut bukan bagian dari rute aktif; bersihkan secara hati-hati bila melakukan refactor.
- Sebelum mengubah dependency, jalankan `php artisan test` dan build aset untuk memastikan tidak ada regresi.
- Jangan menambahkan kredensial atau data pribadi ke dalam repositori.
