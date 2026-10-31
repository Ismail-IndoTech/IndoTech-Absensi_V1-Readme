# Feature & Workflow Guide — IndoTech-Absensi V1

This document explains the application's modules, business rules, and main routes.

---

## 1. Roles & Authentication

The application uses two separate guards in `config/auth.php`:

| Guard | Provider | Model | Table | Usage |
|-------|----------|-------|-------|-------|
| `admin` | `users` | `App\Models\User` | `users` | Administrator panel |
| `employee` | `employees` | `App\Models\Employee` | `employees` | Employee panel |

- Single login at `/login`; the system detects whether the email belongs to an administrator or an employee.
- Administrator sessions last 60 minutes and end when the browser is closed.
- Employee sessions last a long time (1 year) for convenience on attendance devices.
- Accounts with `status != 0` are considered inactive and are rejected at login.
- Employees using an Android browser without the `IndoTech-APP`/`IndoTech-Browser` marker will be directed to download the application.

---

## 2. Administrator Panel

Routes are under the `/admin` prefix with the `auth:admin` middleware.

### 2.1 Reports
- `/admin/reportPresensi` — employee attendance report.
- `/admin/perizinan` — list of employee requests.
- `/admin/permissions` — request processing/approval.

### 2.2 Master Data
| Module | Main route | Description |
|--------|------------|-------------|
| Attendance location | `/admin/lokasi` | Office point, latitude/longitude, radius, accuracy. |
| Radius | via location | Geofence radius per location. |
| Fixed schedule | `/admin/schedule/list` | Work schedules along with patterns (`schedule_patterns`). |
| Shift | `/admin/shift/list` | Work shifts, shift patterns, and shift groups. |
| Division | `/admin/devisi` | Division data and member details. |
| Position | `/admin/positions` | List of positions and sub-positions. |
| Employee | `/admin/employee` | Add, edit, deactivate, and delete employees. |
| Holiday | `/admin/holiday` | Holiday calendar. |
| Admin user | `/admin/user` | Administrator account management. |

### 2.3 Employees
- **Add employee** through `/admin/employee/new`.
- **Bulk import** from a spreadsheet using the `Admin\Dashboard\ImportDataExel` component and the classes in `app/Imports`.
- **Reset device binding** through `PUT /admin/employee/rebind-employee/{id}`.
- **Change status** active/inactive through `PUT /admin/employee/{id}/update-status`.

---

## 3. Employee Panel

Routes with the `auth:employee` middleware.

| Page | Route | Function |
|------|-------|----------|
| Dashboard | `/dashboard` | Attendance summary and today's schedule. |
| Attendance | `/presensi` | Clock-in/clock-out with camera, map, and face detection. |
| Overtime attendance | `/presensiOvTm` | Attendance for overtime activities. |
| History | `/history` | Employee attendance history. |
| Schedule | `/schedule` | Applicable work schedule. |
| Requests | `/requested` | List of requests and their status. |
| Add request | `/requested/add` | Leave, overtime, or schedule change form. |
| Profile | `/profile` | Employee profile data. |

---

## 4. Attendance Rules

The main logic is in `App\Http\Controllers\UserController::absensi()`.

### 4.1 Determining the attendance type
- If there is no `attd_in_time` today → **clock-in**.
- If `attd_in_time` already exists → **clock-out**.

### 4.2 Time validation
| Rule | Value | Response |
|------|-------|----------|
| Check-in too early | More than 45 minutes before the start time | Rejected (code `10`) |
| Check-out too soon | Less than 30 minutes after check-in (or before the end time on certain shifts) | Rejected (code `11`) |
| Past the attendance limit | More than 8 hours after the schedule ends | Rejected (code `12`) |

### 4.3 Cross-day shifts
- If the schedule pattern has `next_day`, the end time is calculated on the following day.
- The system also checks yesterday's schedule: if the current time is still within yesterday's shift range and no check-out has been made, attendance follows that schedule.
- An additional 8-hour tolerance is granted only if the employee has not yet checked out.

### 4.4 Storing attendance data
- Photos are sent as base64, decoded, then saved to the `public` disk with the name `dd-mm-yyyy-<type>.jpg`.
- Data is stored in the `attendances` table: `attd_in_time`, `attd_out_time`, `location_in`, `location_out`, `pict_in`, `pict_out`, `status_in`, `status_out`, `type`, and the schedule/shift/request relations.

---

## 5. Location & Face Validation

### 5.1 Geofencing
- The map uses **Leaflet** with **OpenStreetMap** tiles.
- The office location and radius are taken from the `attend_locations` and `radii` master data.
- The user's position is monitored with `navigator.geolocation.watchPosition`.
- The attendance button is disabled if the user is outside the radius or the GPS accuracy is insufficient.

### 5.2 Face detection
- Uses `face-api.js` with the `tiny_face_detector` model from `public/models`.
- A face must be detected and within a certain size before the attendance button becomes active.
- Detection runs in the browser; final validation is still performed on the server.

### 5.3 Mock location indication
- The attendance page checks the `mockLocation` parameter; if indicated, the attendance button remains disabled.

---

## 6. Scheduling

### 6.1 Fixed schedules
- Tables `schedules` and `schedule_patterns`.
- Patterns determine workdays, start/end times, and the `next_day` marker.

### 6.2 Shifts
- Tables `shifts`, `shift_patterns`, and `shift_groups`.
- Used for rotating work patterns.

### 6.3 Change requests
- **Schedule changes** are stored in `schedule_requests`.
- **Shift swaps** are stored in `shift_requests`.
- Both have an approval status and can affect attendance (`sched_req_id`, `shift_req_id`).

---

## 7. Employee Requests

Request types recognized by the system include:

- `sick` — sick
- `leave` — time off/leave
- `overtime` — scheduled overtime
- `overtimeNotSched` — overtime outside the schedule
- `otgo` — special overtime

Approved requests can produce attendance entries of type `requested`, `changeSched`, `overtime`, or `overtimeNotSched`.

---

## 8. Holidays

- Data in the `holidays` table.
- Used to mark holidays in attendance calculations and schedule views.

---

## 9. Notifications

- The `notifications` table stores `user_id`, `type`, `message`, `data` (JSON), and `read`.
- API endpoints available:
  - `GET /api/notifications`
  - `POST /api/notifications/{id}/read`
  - `POST /api/notifications/{id}/respond`

---

## 10. Audit Log

- The `AppLog` model records important activities to the `app_logs` table.
- Example events: failed login, successful login with a new UUID, device rejection, and attendance rejection due to time rules.
- Use this log for investigation and operational audits.

---

## 11. Device Security

- **Device UUID**: generated on the client and stored in `localStorage`; the first value is saved to the employee's `uuid` column. Login from a different UUID is rejected.
- **Version check**: the `CheckAppVersion` middleware rejects applications below the minimum version (HTTP 426).
- **Login limiting**: 3 failures per email → 60-second block; 5 different emails from one IP → 15-minute block.
- **Root detection**: the `rootDetected=true` parameter at login is directed to special handling.
- **Session check**: `AppServiceProvider` checks `session_expiration` and logs the user out if the session has expired.

---

## 12. Development Notes

- The `tests/` folder still contains sample tests; add feature tests for critical flows (login, attendance, requests).
- Some files have copies marked `(1)` (e.g., `Dashboard(1).php`). Those files are not part of the active routes; clean them up carefully when refactoring.
- Before changing dependencies, run `php artisan test` and build the assets to ensure there are no regressions.
- Do not add credentials or personal data to the repository.
