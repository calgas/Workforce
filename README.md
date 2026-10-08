# 🏢 CALGAS Workforce — HR & AI Attendance System

> **Zero-server, AI-powered factory attendance, payroll, and HR management system.**
> Built on Google Sheets + Google Apps Script + a PWA frontend for CALGAS Capacitors
> (CALGAS Mobility Pvt. Ltd.), Navsari, Gujarat.
> No recurring hosting costs. No build pipeline. No dependencies to install.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture](#2-architecture)
3. [File Structure](#3-file-structure)
4. [Feature Reference](#4-feature-reference)
5. [User Roles](#5-user-roles)
6. [Google Sheet Structure](#6-google-sheet-structure)
7. [API Keys, Secrets & Mail — How to Configure](#7-api-keys-secrets--mail--how-to-configure)
8. [Setup & Deployment](#8-setup--deployment)
9. [Deployment Checklist](#9-deployment-checklist)
10. [Offline & Sync Behaviour](#10-offline--sync-behaviour)
11. [AI Face Recognition](#11-ai-face-recognition)
12. [Payroll Engine Logic](#12-payroll-engine-logic)
13. [Security Model](#13-security-model)
14. [Version History](#14-version-history)
15. [Known Limitations](#15-known-limitations)
16. [Troubleshooting](#16-troubleshooting)

---

## 1. System Overview

**CALGAS Workforce** is a **Progressive Web App (PWA)** that runs entirely on Google infrastructure with zero recurring server costs. Designed for the factory floor at CALGAS Capacitors, Navsari, it replaces manual attendance registers and spreadsheet payroll with:

- **Instant AI facial recognition** with 2-of-3 frame confirmation and live canvas bounding-box tracking for fraud-proof punch IN / OUT at a shared kiosk.
- **True offline-first architecture** using IndexedDB and the Background Sync API so factory floor punches are never lost, even if the browser tab is closed.
- **Automated payroll engine** that calculates prorated salary, overtime, ESI, PF, VPF, Gujarat Professional Tax, comp-off (SOT), and a Year-to-Date summary — with the sandwich rule and VPF applied identically in the individual payslip and the Excel salary report.
- **Native PDF payslips** on CALGAS letterhead with an editable, additive Advance (ADV.) field and optional Year-to-Date section, printable directly from the app with crisp, selectable vector text.
- **Encrypted biometric data** — face descriptors are encrypted at rest, not stored as plain text.
- **Automated nightly backups** of the entire database to Google Drive, independent of any manual process.
- **Multi-tier role access** — Admin, HR, Security (numbered shifts), Standby kiosk, and Employee self-service — with self-service password reset (mailed from noreply@calgas.in) and brute-force login protection.
- **Live dashboard** featuring category-wise attendance breakdown, visual enrolled-staff avatars, and instant Excel matrix exports.
- **History in table format**, filterable by employee and date, with server-side filtering so narrowing to one person is as fast as viewing everyone.
- **Employee correction requests** — a full workflow for employees to flag incorrect attendance for HR/Admin review.
- **In-app Help Guide** on every screen — no separate training manual required.

```text
┌─────────────────────────────────────────────┐
│              FACTORY FLOOR                  │
│  Shared Tablet (Standby / Kiosk role)       │
│  2-of-3 Frame Face Recognition + Box Track  │
│  Auto-Punch IN / OUT (No blink required)    │
└────────────────┬────────────────────────────┘
                 │  HTTPS POST (JSON)
                 ▼
┌─────────────────────────────────────────────┐
│         Google Apps Script (Code.gs)        │
│  REST API · LockService · SHA-256 Auth      │
│  Rate Limiter · Script Properties (SECRET)  │
│  Face Data Encryption · Nightly Backups     │
│  Mail as noreply@calgas.in                  │
└────────────────┬────────────────────────────┘
                 │  SpreadsheetApp read/write
                 ▼
┌─────────────────────────────────────────────┐
│            Google Sheets (8 tabs)           │
│  Data · Users · List_of_Empl · Shifts       │
│  Audit_Log · HS · OT_Empl · List of Holidays│
└─────────────────────────────────────────────┘
```

---

## 2. Architecture

| Layer | Technology | Purpose |
|---|---|---|
| Frontend | HTML / CSS / Vanilla JS | Single-file PWA — no build step, responsive mobile-first layout with full light/dark theme support in CALGAS indigo (`#3d3f94`, taken from the logo). |
| Typography | Montserrat (self-hosted variable font) | One 34 KB `woff2` file covers weights 100–900 for the whole app, the printed payslip and the offline page. Served from the repo and cached by the Service Worker, so it works offline. |
| AI | face-api.js (TinyFaceDetector) | Client-side facial recognition — processes instantly via WebGL. Zero cloud calls. |
| Backend | Google Apps Script (`doPost`) | REST-like API with script locking, session tokens, rate limiting, chunked cache, and biometric encryption. |
| Database | Google Sheets (8 tabs) | Zero-cost persistent storage with ArrayFormulas for hours and overtime. |
| Offline | Service Worker + IndexedDB | Stale-while-revalidate shell cache; pending punches background-sync with a 25s timeout guard. |
| Export | ExcelJS + native `window.print()` | Frozen-pane Excel attendance matrices and salary reports, plus crisp PDF payslips with an editable Advance field and Year-to-Date summaries. |
| Secrets | Google Script Properties | `APP_SECRET` stored server-side — signs session tokens **and** derives the biometric encryption keystream. Hard-fails at login if missing, never exposed to the browser. |
| Mail | `GmailApp` via `sendMail_()` | All outbound mail sends as `noreply@calgas.in` when that address is a verified alias on the script owner's account; falls back to the owner's address otherwise. |
| Backup | Google Drive | `backupAllSheets()` copies core sheets into a dated spreadsheet every night via a time-driven trigger; 30-day auto-pruned retention. |

---

## 3. File Structure

```text
Workforce/
├── index.html                      # Entire frontend — UI, styles, AI models, canvas overlays, and JS logic
├── manifest.json                   # PWA manifest — relative paths, local icons, shortcuts
├── sw.js                           # Service Worker — auto-versioned cache, offline fallback, background sync
├── fonts/
│   ├── montserrat-latin-var.woff2  # Montserrat variable font (weights 100–900), subset to Latin + ₹
│   └── OFL.txt                     # SIL Open Font License for Montserrat
├── logo/                           # Logo files served from the repo (same set as Stock Management)
│   ├── icon-192.png
│   ├── icon-512.png
│   ├── icon-512-maskable.png
│   └── CALGAS CAPACITORS-logo-768x240.jpg
├── Code.gs                         # Google Apps Script backend — paste into the Sheet's Apps Script editor
├── CALGAS_Workforce_Template.xlsx  # Fresh 8-tab Google Sheet template (+ Setup tab with instructions)
├── calgas-presentation.html        # Standalone system presentation
└── README.md
```

> Only `index.html`, `manifest.json`, `sw.js`, and the `fonts/` and `logo/` folders are hosted on GitHub Pages.
> `Code.gs` lives in the Google Sheet's Apps Script project; the template and presentation are reference files.
>
> **Hosting:** `github.com/calgas/Workforce` → `https://calgas.github.io/Workforce/`.
> `manifest.json` uses relative `./` paths for `start_url`, `scope`, icons and shortcuts, so the app
> works under any repository name without edits.
>
> **Shared origin:** the CALGAS apps (Stock Management, ELE Tracker, Production Tracker, this app)
> all live on `calgas.github.io`, and browsers keep Cache Storage, IndexedDB and `localStorage`
> per origin, not per app. Every storage name in this app is therefore its own:
> cache prefix `calgas-workforce-`, databases `CalgasWorkforceDB` and `CalgasWorkforceSWMeta`,
> and `localStorage` keys `wf_session` and `wf_theme`. Cache cleanup only deletes caches with
> this app's prefix. A new sibling app must pick names of its own.

---

## 4. Feature Reference

### 4.1 Manual Attendance Entry
- HR / Admin searches for an employee using a debounced, lag-free live-filter dropdown.
- Selects **Punch IN**, **Punch OUT**, **Mark Permission**, or **Mark Leave**.
- Leave type dialog: **EL** (Earned Leave — deducts balance) or **LOP** (Loss of Pay — no deduction). Selected type is preserved in offline punch payloads and survives background sync.
- Leave confirmation shows the employee's exact current leave balance before committing.
- Optional free-text remarks field for context.
- Live status badge queries the server instantly to prevent duplicate punches.
- When an employee already has a punch-in today, the shift selector is pre-populated with their active shift — HR does not need to guess.
- After a successful punch, only the affected employee's status badge refreshes — no full data reload.
- Action buttons are disabled during the API call to prevent accidental double-submit.

### 4.2 AI Face Recognition Kiosk (2-of-3 Frame Confirmation)
- **Standby role** devices display only the kiosk UI — no access to admin functions.
- Admin selects IN or OUT; camera activates (with mobile autoplay and `webkit-playsinline` bypasses).
- **2-of-3 Frame Confirmation:** Requires the same employee to be matched in at least 2 consecutive frames before logging the punch. Eliminates false positives from look-alikes or partial faces passing the camera.
- **inputSize 160** (not 320) — 4× faster GPU inference on mobile. Frame confirmation compensates for the lower resolution by requiring consensus.
- **Live Canvas Tracking:** A bounding box draws in real time over the detected face with the employee's name.
- Text-to-speech announces the employee's name to confirm.
- **120-second countdown timer** visible live so the kiosk operator knows when it auto-closes.
- The employee's `category` is passed to the server at punch time — the backend skips a redundant `getEmployees()` call for the SOT bonus check.
- Face descriptors are fetched via the dedicated `getKioskFaceData` action (decrypted server-side, cached 15 minutes) rather than the general employee list — see [4.10](#410-data-protection--encrypted-biometrics--automated-backups).
- **Kiosk punches are never dropped:** if a recognised punch can't reach the server (device offline, or Wi-Fi with no internet), it is saved to the offline queue with the time the face was recognised and syncs later. If the server simply takes too long to answer, the kiosk says "not confirmed — check History" instead of queuing a second copy.
- The installed app's **Punch IN / Punch OUT** shortcuts open straight into the kiosk (`./index.html?action=in|out`).

### 4.3 Face Enrollment
- Admin finds the employee card and taps **Enroll Face** (or **🔄 Re-enroll** if already enrolled).
- Live canvas tracks the face to confirm good framing and lighting.
- Camera captures **5 distinct 128-float descriptors** (higher `inputSize: 320` for maximum accuracy).
- Progress dots update in real time showing capture count (`1/5`, `2/5`, …).
- Descriptor array is JSON-serialised, **encrypted**, and saved to column Q of `List_of_Empl`.
- After enrollment, `FaceMatcher` is immediately rebuilt using the plaintext just captured — no round trip needed to re-fetch and decrypt what the browser already has.

### 4.4 True Offline Background Sync
- If the factory loses internet, punches are saved directly to the browser's **IndexedDB** (`CalgasWorkforceDB`) — including `leaveType` (EL/LOP) and `shift` so all fields survive sync faithfully.
- **Wi-Fi without internet** counts as offline too: the browser still reports "online", so a punch whose request fails before reaching the server is queued the same way, from both manual entry and the kiosk. A request that times out (it may already have reached the server) is reported as "not confirmed" rather than queued twice.
- The Service Worker stays out of the API path entirely — API calls go straight from the page, so the page always sees the real outcome.
- The Service Worker registers a `sync-punches` tag with the OS.
- When the OS detects Wi-Fi, it silently POSTs all pending punches to `Code.gs` with a **25-second AbortController timeout** to prevent stalled sync events.
- Backend deduplicates via fingerprint (`emplId|date|time|action`) — zero duplicate rows.
- Punches are grouped by session token before sending so multi-user offline sessions authenticate independently.
- **iOS Safari fallback:** Background Sync API is unsupported on iOS. An `online` event listener flushes IDB directly when the app is foregrounded and reconnects.

### 4.5 Attendance History (Table Format)
- Filter by **month**, **exact date**, **date range**, and/or a **multi-select employee filter**.
- Results always render as a **table** — Date, Shift, In, Out, Permission, OT, and an Edit/Req action column. When more than one employee is in the result set, an Employee column appears too; single-employee results show the name once in the header instead.
- **Server-side filtering:** selecting specific employees is filtered on the backend before the response is built, so narrowing to one person is as fast as viewing everyone.
- **CSV Export:** Converts the exact on-screen filtered view — including the employee filter — directly to a downloadable CSV.
- **Employee Correction Requests:** Employees click **Req** on any history row to open a modal describing the discrepancy. The request is written to `Audit_Log` as a `PENDING CORRECTION` entry. HR/Admin resolve it with a single button that stamps `RESOLVED`.

### 4.6 Salary Engine & PDF Payslips
- Full payroll logic including ghost-Sunday-proof calendar mathematics and the sandwich rule (see Section 12).
- **Comp-Off (CO) tracking:** SOT employees who work ≥ 12 hours receive a 0.5-day leave credit, flagged as `SOT_BONUS_ADDED` in column O. Monthly Excel reports show the real per-employee CO total.
- **Editable Advance (ADV.):** The payslip shows Net Payout, then an editable ADV. field, then a Final Payable amount (Net + ADV.) — ADV. is additive, not a deduction. The value carries from the in-app view into the printable PDF automatically.
- **Print inclusion toggles:** Two checkboxes — "Include ADV. section" and "Include Year-to-Date Summary" — control what actually appears in the printed/saved PDF. Unchecking both strips them from the print output entirely, not just visually.
- **Year-to-Date Summary:** Cumulative gross, deductions, net pay, OT hours, present days, and leaves availed from the start of the financial year (April 1). Computed by summing each elapsed month's own `getEmpDashData()` result, so the YTD total always matches what each individual month's payslip already showed. Cached 15 minutes per employee/month.
- **Attendance % uses working days** (not total month days) as the denominator — e.g. 22 present out of 22 working days = 100%, not 22/30 = 73%.
- **Letterhead:** The printed payslip carries the CALGAS wordmark, "CALGAS Mobility Pvt. Ltd.", and the Navsari plant address (Plot No. 61, Rajhans Zesto, NH-8, Vesma, Navsari, Gujarat – 396415).
- **Native PDF Printing:** A visible **Print / Save as PDF** button inside the payslip lets the person review the carried-over ADV. value and the included sections first. `@media print` CSS strips the app chrome, leaving only the payslip.

### 4.7 Monthly Excel Salary Report
- Downloads a formatted `.xlsx` file titled `CALGAS CAPACITORS | <MONTH> <YEAR> SALARY STATEMENT`, one row per selected employee.
- **Raw data columns:** Employee ID, Name, Gross, Basic, HRA, Conv, Spl Allow, Medical, **VPF**, P/Days, Ph/Days, Leaves Availed, **Sandwich Ded.**, Absent Days, Total Days.
- **Formula columns:** Prorated Basic/HRA/Conv/Spl/Medical/Gross, EPF, ESI, PT, VPF, Total Deductions, Net, **ADV. (manual, additive)**, **Final Payable (Net + ADV.)**, Signature.
- **PT column uses the Gujarat slab** (`=IF(Gross>12000,200,0)`) — the same rule as `professionalTax()` in `Code.gs`, so the report and the payslip agree.
- The sandwich rule and VPF are applied here exactly as in the individual payslip — Payable Days in every prorated formula subtracts the Sandwich Ded. column.
- Column styles are applied once at the column level; row values are written in bulk — avoids per-cell style objects that would hang the browser on large exports.
- **Cutoff logic:** For the current month, only data up to today is counted. Future days show blank cells (not "Absent"). Past months always use the full month.
- Supports partial-employee export by selection.

### 4.8 Live Admin Dashboard
- Live total staff, present, currently in, out, on leave, and late arrival counts.
- **Present by Category:** A live progress bar per staff category (Staff Without OT / With OT / With SOT) showing "X / Y present" — surfaces which workforce segment is short-staffed at a glance.
- **Enrolled Avatars:** Horizontally scrolling circular avatars (initials) for every enrolled employee.
- **Monthly Attendance Matrix (Excel):** Frozen-pane matrix with `X` (present), `EL` (leave), `A` (absent), summary columns (P, H, PD, EL, CO, H/S, A, TPD, PST EL, AVAIL EL) per employee.
- The refresh button shows a loading spinner and an error toast on failure, consistent with every other data-fetch action in the app.

### 4.9 Admin Panel (Inventory)
- **Users Management:** Create, edit, and delete app login accounts. Assign roles — Admin, HR, Standby, **Security-1**, **Security-2**, or Employee — and link Employee-role accounts to their `Empl_ID` (column E).
- **Employee Master:** Full CRUD for employee records including all salary components, shift assignment, category, leave balance, and PIN.
- **Bulk Employee Import:** Paste CSV to create multiple employees in one shot (duplicate ID check included).
- **Leave Management:** Assign single or multi-day EL or LOP leave ranges. Skips Sundays and public holidays automatically. A **visual calendar grid** (up to 3 months) renders below the date pickers, highlighting exactly which days will be marked before submitting — Sundays within the range shown muted. Processed in a single optimised sheet pass — no per-day re-read.
- **Correction Requests Panel:** View, review, and resolve employee attendance correction requests.
- **Attendance History Edit:** Admin can update IN time, OUT time, Permission time, and Remarks on any historical record. All edits are logged to `Audit_Log` with the before and after values.

### 4.10 Data Protection — Encrypted Biometrics & Automated Backups
- **Face data encryption:** Face descriptors are encrypted at rest in column Q of `List_of_Empl` using a stream cipher — HMAC-SHA256(`APP_SECRET`, hex IV + counter) generates a keystream XORed against the plaintext, stored as `ENC1:<hex IV>:<base64 ciphertext>`. A random IV per save means re-saving identical data produces a different ciphertext each time. The HMAC input is a **string** (hex IV + counter), not a raw byte array — Apps Script's JS→Java bridge does not reliably recognize a plain array as a native `byte[]`.
- Legacy unencrypted values pass through unchanged for backward compatibility; `migrateEncryptAllFaceData()` encrypts everything already on the sheet. A fresh CALGAS sheet starts empty and encrypted from the first enrollment, so it is only needed if face data is ever imported from elsewhere.
- This is a lightweight, dependency-free cipher — Apps Script has no native AES. It protects data from casual spreadsheet access; a stronger cryptographic guarantee would require routing through an external KMS.
- **Automated nightly backups:** `backupAllSheets()` copies `Data`, `Users`, and `List_of_Empl` into a dated spreadsheet inside a **"CALGAS Workforce Backups"** Drive folder every night at 2 AM IST. Backups older than 30 days are automatically deleted. Install the schedule once with `setupNightlyBackupTrigger()`.

### 4.11 Self-Service Password Reset
- A "Forgot password?" link on the login screen opens a two-step flow: enter a username to receive a 6-digit code by email, then submit the code with a new password.
- Codes expire after 15 minutes and are single-use. Requests are capped at 3 per hour per username to prevent inbox flooding.
- The server returns an identical, generic response whether or not the username exists — the flow cannot be used to enumerate valid logins.
- A successful reset clears any active login lockout on that account and appends a `Password Reset` entry to `Audit_Log`.
- Mail is sent by `sendMail_()` as **noreply@calgas.in** with sender name "CALGAS Workforce" — see [7.4](#74-outbound-mail--noreplycalgasin).

### 4.12 In-App Help Guide
- A slide-down panel (not a full-screen modal) — opens beneath the header on tap, dismissible by tapping outside or the trigger icon again, matching the notification bell's interaction pattern.
- Content is scoped per screen and per admin/report sub-tab, including a dedicated entry for the Dashboard tab.
- No separate training manual needed for new HR, Security, or Standby-kiosk staff.

---

## 5. User Roles

| Role | Entry | History | Payslip | Correction Request | Admin Panel | Kiosk |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **Admin** | ✅ | ✅ All | — | ✅ Resolve | ✅ Full | — |
| **HR** | ✅ | ✅ All | — | ✅ Resolve | ✅ No user mgmt | — |
| **Security-1 / Security-2** | ✅ | ✅ All | — | — | — | — |
| **Employee** | — | ✅ Own only | ✅ Own | ✅ Submit | — | — |
| **Standby** | — | — | — | — | — | ✅ Only |

> Role names are **case-sensitive**. Type them exactly as shown in the Users sheet — Security
> roles specifically must be `Security-1` or `Security-2`, the two numbered variants the app
> recognizes (intended for separate guard shifts/checkpoints, not a single shared role).
> All roles are selectable directly from the Admin Panel's Create/Edit User dropdown.
>
> This app keeps its own login, separate from the other CALGAS apps. The two-layer
> Admin/department access model used by Stock Management is not applied here yet.

---

## 6. Google Sheet Structure

Start from **`CALGAS_Workforce_Template.xlsx`** — it has all 8 tabs with headers, number formats,
the starter rows below, and a **Setup** tab with step-by-step instructions (the backend never reads
the Setup tab; delete it once you're done). See [Section 8](#8-setup--deployment).

### Tab: `Data` (Attendance Log)
*Columns A–O:* Date, Day, Empl ID, Name, Shift, Shift Start, IN Time, Shift End, OUT Time, Tot. Hrs (ArrayFormula), OT Hrs (ArrayFormula), Permission, Remarks, Logged By, Flags.

> Columns J and K are driven by `ARRAYFORMULA` in row 1. Excel can't store these Google-only
> formulas, so the template keeps them as text in the Setup tab — paste them into **J1** and **K1**
> after importing. They encode the current attendance rules:
>
> - **J — hours worked:** OUT − IN, minus a 30-minute lunch when IN ≤ 13:30 and OUT ≥ 14:00.
> - **K — overtime:** counted in 30-minute blocks (5-minute grace) after 8 h when the day ran
>   ≥ 12 h, otherwise after 8.5 h; on Sundays and listed holidays, everything beyond a 30-minute
>   break counts.
>
> Both use open-ended ranges (`G2:G`), so they never stop at a fixed row. The backend uses
> `getInsertRow()` to find the first genuinely empty row in Column C — bypassing ghost rows
> created by the formulas. Column O (Flags) stores internal markers: `SOT_BONUS_ADDED`
> (CO tracking) and `LOP`.

### Tab: `List_of_Empl` (Employee Master)
*Columns A–Q:* ID, Name, Shift, Category, Leave Bal, Gross, Basic, HRA, Conv, Spl Allow, Med, ESI, PF, VPF, PT, PIN, **Face Data (encrypted)**.

> Category must be one of `Staff Without OT`, `Staff With OT`, `Staff With SOT`.
> For a row typed by hand, the per-row formulas are `L: =IF(F2<=21000,ROUND(F2*0.0075,0),0)`,
> `M: =ROUND(G2*0.12,0)`, `O: =IF(F2>12000,200,0)` (Gujarat PT). These columns are for reading the
> sheet — payroll always recomputes ESI, PF and PT itself. Rows added from the app store the
> values shown in the form.
>
> **Column Q — Face Data:** Stored as `ENC1:<hex IV>:<base64 ciphertext>` (see [4.10](#410-data-protection--encrypted-biometrics--automated-backups)). Never edit this column by hand.

### Tab: `Users` (App Logins)
*Columns A–E:* Username, Role, Email, Password (SHA-256), **Empl_ID**.

> **Column B — Role:** One of `Admin`, `HR`, `Standby`, `Security-1`, `Security-2`, or `Employee`.
>
> **Column C — Email:** Required for that account's self-service password reset (Section 4.11). If blank, reset requests for that username silently do nothing — by design, so the response can't be used to confirm the account exists.
>
> **Column D — Password:** Type a plain-text password for a new row; it is replaced by its SHA-256 hash on the first successful login. A row with a blank password can never sign in.
>
> **Column E — Empl_ID:** For **Employee-role accounts**, enter the matching employee ID
> from `List_of_Empl` column A. Leave blank for Admin, HR, Security, and Standby accounts.
>
> The template ships with one starter Admin row (`Kanna`, `Admin`) — fill in its Email and Password.

### Tab: `Shifts`
*Columns A–D:* Shift Name, Shift Start, Shift End, OT Hours Threshold.

> The template has one starter `REGULAR` row (09:00–17:30). Replace the timings with CALGAS
> shifts and add the others. Keep a shift named `REGULAR` — the backend falls back to it when an
> employee has no shift.

### Tab: `HS` (Month Settings)
*Columns A–C:* Month Name, No. of Days, PH Count. Overrides the calendar day count used for payslip proration in a given month. Column C is for reference — holidays are counted from `List of Holidays`.

> Named `HS` (no slash) because Excel cannot hold "/" in a tab name, so `H/S` would not survive
> an `.xlsx` round-trip. The backend reads `HS` and still accepts an older `H/S` tab.

### Tab: `OT_Empl` (OT Gross Overrides)
*Columns A–C:* Empl ID, Name, OT Gross. Overrides an employee's standard gross for OT rate calculations.

### Tab: `List of Holidays`
*Columns A–B:* Date, Reason. Used by payroll, leave marking, the overtime formula and the sandwich rule to identify public holidays.

> Pre-filled with every Sunday of 2026–2027 and the three national holidays (Republic Day,
> Independence Day, Gandhi Jayanti). Add CALGAS festival holidays as `dd-mm-yyyy` dates.

### Tab: `Audit_Log` (Immutable History)
*Columns A–F:* Timestamp, Actor (Admin/User), Empl ID, Empl Name, Old Value / Action Type, New Value / Details. All admin edits, user deletions, employee deletions, password resets, and correction requests are appended here.

### Drive: "CALGAS Workforce Backups" Folder
Not a sheet tab — a separate Google Drive folder, created automatically on first backup run. Contains one dated spreadsheet per night (`Backup_YYYY-MM-DD`), each with copies of `Data`, `Users`, and `List_of_Empl`. Entries older than 30 days are auto-deleted.

---

## 7. API Keys, Secrets & Mail — How to Configure

### 7.1 `GOOGLE_API_URL` (Apps Script Web App URL)
This HTTPS endpoint connects the PWA to the Google Sheet. It ships as the placeholder
`YOUR_GOOGLE_SCRIPT_WEB_APP_URL_HERE` — the app refuses to log in until it is set. Update it in
**two places** after every new deployment:
- **`index.html`** — `const GOOGLE_API_URL = "..."` near the top of the main `<script>` block.
- **`sw.js`** — `const GOOGLE_API_URL = "..."` at the very top of the file.

### 7.2 `APP_SECRET` (Session Signing Key **and** Biometric Encryption Key)
This is a **true secret** — never place it in the frontend. It serves two purposes:
1. Signs session tokens.
2. Derives the keystream used to encrypt and decrypt face descriptor data (Section 4.10).

The backend **hard-fails** on login and on any face-data operation if `APP_SECRET` is not set. There is no insecure fallback.

1. In the Apps Script editor → **⚙️ Project Settings** → **Script Properties** → **Add script property**
2. Property name: `APP_SECRET`
3. Value: *a new long random string — minimum 32 characters, generated fresh for CALGAS*

> ⚠️ If login returns `"APP_SECRET script property is not configured"`, this step was skipped.
> ⚠️ **Never rotate `APP_SECRET` on a live sheet without a plan** — changing it invalidates the ability to decrypt any face data encrypted under the old value. Re-enroll affected employees, or decrypt-then-re-encrypt under the new secret first.

### 7.3 `app-version` Meta Tag (Cache Auto-Busting)
The Service Worker reads the cache version from `index.html` at install time — no manual constant to update in `sw.js`.

Update this line in `index.html` on every deploy:
```html
<meta name="app-version" content="20261007">
```
Format: `YYYYMMDD`, optionally with a same-day revision letter suffix (e.g. `20261007b`) for a second deploy on the same date — the value is used as an opaque cache-key string, not parsed as a date. Changing it causes all users to receive the fresh build on their next visit.

### 7.4 Outbound Mail — noreply@calgas.in
All mail (currently the password-reset code) goes through `sendMail_()` in `Code.gs`, sending as
`MAIL_FROM` (`noreply@calgas.in`) with sender name `MAIL_SENDER_NAME` ("CALGAS Workforce") and
`replyTo` set to the same address.

Apps Script **cannot send as an arbitrary address**. `GmailApp.sendEmail` accepts a `from` only when
that address is a verified *Send mail as* alias on the Google account that owns the script (or is
that account itself). So:

1. On the account that owns this Apps Script project, go to **Gmail → Settings → Accounts → Send mail as** and add `noreply@calgas.in`, completing verification. (Skip if the script is owned by the `noreply@calgas.in` account itself.)
2. Run **`checkMailSender()`** once from the Apps Script editor (▶ Run). It triggers the Gmail authorization prompt and logs which address mail will actually be sent from.

If the alias is missing, mail **still sends** under the owner's own address rather than failing —
a misconfigured alias must never swallow a password-reset code. Sending counts against the owner
account's daily Gmail quota (calgas.in is on Google Workspace, so the quota is far above a personal
Gmail account's).

---

## 8. Setup & Deployment

Set this up on the CALGAS Google Workspace account. Start from a **fresh sheet** — no employees,
users, attendance or face data are carried over from any earlier deployment.

1. Upload **`CALGAS_Workforce_Template.xlsx`** to Google Drive → open it → **File → Save as Google Sheets**. Work only in the Google Sheets copy.
2. In the Google Sheet: **File → Settings → Locale: India, Time zone: (GMT+05:30) India Standard Time**. The backend writes dates as `dd-mm-yyyy`; under a US locale `07-10-2026` would be read as 10 July.
3. Paste the two formulas from the **Setup** tab into **Data!J1** and **Data!K1** (Section 6).
4. **Users** tab, row 2: fill in the starter Admin's **Email** and **Password**.
5. **Shifts** tab: replace the starter `REGULAR` timings with CALGAS shifts. **List of Holidays**: add CALGAS festival holidays.
6. **Extensions → Apps Script** → paste `Code.gs`.
7. **⚙️ Project Settings → Script Properties** → add a new `APP_SECRET` (Section 7.2).
8. **Deploy → New Deployment** (Type: Web App, Execute As: Me, Who has access: Anyone).
9. Copy the Web App URL → paste into `GOOGLE_API_URL` in both `index.html` and `sw.js`.
10. Add the `noreply@calgas.in` alias if needed, then run **`checkMailSender()`** once (Section 7.4).
11. **One-time:** Run **`setupNightlyBackupTrigger()`** from the Apps Script editor (▶ Run) to install the 2 AM IST nightly backup schedule. Safe to re-run — existing triggers for this function are replaced, not duplicated.
12. Create the **`calgas/Workforce`** GitHub repository. Upload `index.html`, `manifest.json`, `sw.js`, and the whole **`fonts/`** and **`logo/`** folders (the Service Worker caches these files at install, so a missing file stops it installing). Settings → Pages → deploy from the `main` branch.
13. Update `<meta name="app-version" content="YYYYMMDD">` in `index.html` to the deploy date.
14. Open `https://calgas.github.io/Workforce/`, sign in as the Admin, then add HR / Security / Standby / Employee accounts and employees from **Inventory**. Enroll faces from the Employees list.

> `migrateEncryptAllFaceData()` is not needed on a fresh sheet — every enrollment is encrypted from
> the start. Keep it for the day face data is ever imported from an unencrypted source.

---

## 9. Deployment Checklist

- [ ] Google Sheet locale set to **India** and time zone to **IST**.
- [ ] `ARRAYFORMULA`s pasted into `Data` sheet cells **J1** and **K1**.
- [ ] Starter Admin row in `Users` has an Email and a Password.
- [ ] `APP_SECRET` set in Script Properties (a fresh value, not one from another deployment).
- [ ] Re-deployed Google Apps Script as **New Deployment → Execute As: Me → Anyone**.
- [ ] Updated `GOOGLE_API_URL` in **both** `index.html` and `sw.js` (no placeholder left).
- [ ] `checkMailSender()` logs `✅ Mail will be sent as noreply@calgas.in`.
- [ ] `setupNightlyBackupTrigger()` has been run — Apps Script → Triggers shows a `backupAllSheets` entry.
- [ ] `fonts/` and `logo/` folders uploaded alongside `index.html`, `manifest.json`, `sw.js`.
- [ ] Updated `<meta name="app-version" content="YYYYMMDD">` in `index.html`.
- [ ] Populated **column E (Empl_ID)** in `Users` for all Employee-role accounts.
- [ ] Populated **column C (Email)** in `Users` for accounts that need self-service password reset.

---

## 10. Offline & Sync Behaviour

1. Device goes offline — or the request fails before reaching the server — → the punch is written to **IndexedDB** (`CalgasWorkforceDB`) by `queueOfflinePunch()`, including `leaveType`, `shift`, `loggedByUser`, the original punch time, and the session token so all fields are preserved exactly. Manual entry and the kiosk both use this path.
2. A `sync-punches` tag is registered with the OS via `SyncManager`.
3. When the OS regains Wi-Fi, `sw.js` silently POSTs punches to `Code.gs` with a **25-second AbortController timeout**. Stalled requests time out cleanly and the browser reschedules a retry.
4. Punches are **grouped by session token** before sending — morning employee + afternoon admin punches authenticate independently.
5. `Code.gs` deduplicates via fingerprint (`emplId|date|time|action`) — zero duplicate rows possible.
6. IDB records are deleted after the server confirms sync. Records that fail (e.g. session expired) are kept for the next retry.

> **iOS Safari:** Background Sync API is unsupported. An `online` event listener calls
> `flushPendingPunchesManually()` directly when the app is in the foreground and the
> device reconnects.

---

## 11. AI Face Recognition

### Model Details
Uses **face-api.js v0.22.2** — runs entirely in-browser via WebGL. Zero cloud API calls.
All weight file URLs are pinned to `@0.22.2` in both `sw.js` and `index.html` to prevent silent model-version drift.

| Mode | inputSize | scoreThreshold | Notes |
|---|---|---|---|
| **Kiosk scan** | `160` | `0.3` | 4× faster GPU pass. 2-of-3 confirmation compensates for lower resolution. |
| **Enrollment** | `320` | `0.3` | Higher resolution for maximum descriptor accuracy. |

- **Matching threshold:** `FaceMatcher` distance threshold set to `0.6`.
- **Live tracking:** A `<canvas>` overlay draws a real-time bounding box with the employee's name over the detected face.
- **Data at rest:** Descriptors are encrypted before being written to `List_of_Empl` column Q (Section 4.10). The kiosk fetches decrypted descriptors via the dedicated `getKioskFaceData` action, cached server-side for 15 minutes and invalidated immediately on any new enrollment.

### 2-of-3 Frame Confirmation
The kiosk maintains a `frameHits` counter per employee. Only when the same employee is matched in **2 or more consecutive frames** does the punch fire. All other counters reset to 0 on each frame. Protects against:
- Partial faces at the edge of frame
- Look-alikes walking past the camera
- Low-light single-frame mismatches

### Enrollment Flow
- 5 distinct 128-float descriptors captured per employee (progress dots shown in real time).
- JSON-serialised, encrypted, and written to column Q of `List_of_Empl`.
- `FaceMatcher` is rebuilt immediately using the plaintext just captured in-browser — no round trip to re-fetch and decrypt.

---

## 12. Payroll Engine Logic

```text
Payable Days = Present Days + Public Holidays + Sundays − Sandwich Days
Proration Factor = Payable Days / Total Days in Month

OT Per Hour  = Round(OT Gross / Total Days / 8, 2)
OT Earnings  = Total OT Hours × OT Per Hour

ESI          = Prorated Gross × 0.0075   (only if Gross ≤ ₹21,000)
PF           = Prorated Basic × 0.12
PT (Gujarat) = ₹200 (Gross > ₹12,000) | ₹0     — professionalTax() in Code.gs

Net Pay          = Prorated Gross + OT Earnings − (ESI + PF + VPF + PT)
Final Payable    = Net Pay + ADV.   (ADV. is additive, entered manually, not a deduction)

SOT Comp-Off = +0.5 leave day per shift where elapsed time ≥ 12 hours
               (flagged SOT_BONUS_ADDED in column O of Data sheet)

YTD Summary  = Σ getEmpDashData() for every month from April 1 through
               the requested month (Indian financial year, April–March)
```

### Professional Tax (Gujarat)
Gujarat State Tax on Professions, Trades, Callings and Employments: nil up to ₹12,000 monthly
salary, ₹200 per month above it, assessed on the employee's full monthly gross. The slab lives in
three places that must always change together:

1. `professionalTax()` in `Code.gs` (`PT_EXEMPT_UPTO`, `PT_MONTHLY`) — payslip and YTD.
2. The PT formula column (`X`) in the Excel Salary Report in `index.html`.
3. The per-row PT formula in `List_of_Empl` column O (informational).

### Sandwich Rule
If an employee is absent on a day sandwiched between two non-working days (Sundays or public holidays) and **neither** neighbour worked at least 4 hours, the sandwiched day is not counted as a Payable Day. A 14-day safe search window walks in each direction to find the nearest working day. If no valid working day is found within 14 days in either direction, the check returns `false` — no penalty applied. This rule is applied identically in the individual payslip (`getEmpDashData`), the Monthly Dashboard (`exportMonthlyDashboard`), and the Monthly Salary Report (`exportSalaryReport`), so all three agree on Payable Days for a given employee/month.

### Leave Marking
- **EL (Earned Leave):** Deducts 1.0 from the employee's leave balance. Written as `IN = LEAVE`, `OUT = LEAVE`.
- **LOP (Loss of Pay):** No leave balance deduction. `LOP` flag written to column O of the Data sheet.
- `markLeaveAdmin()` skips Sundays and public holidays, processes the entire date range in a single pre-read pass, and tracks `nextInsertRow` without re-scanning column C on each iteration.
- The frontend renders a visual calendar preview of the selected range before submission (Section 4.9).

### Current Month Cutoff
`exportMonthlyDashboard()`, `exportSalaryReport()`, and payslip generation all detect whether the requested month is the current month. If yes, data is processed only up to today's date — future days show blank (not "Absent"). Past months always use the full month.

### Year-to-Date Summary
`getYTDSummary(emplId, asOfMonthStr)` determines the Indian financial year start (April of the current year, or the previous year if the requested month is Jan–Mar), then sums `getEmpDashData()` across every elapsed month. This guarantees the YTD figures always agree with what each individual month's payslip already displayed — there is no separate, independently-derived calculation path to drift out of sync. Cached 15 minutes per employee/month combination.

---

## 13. Security Model

| Control | Detail |
|---|---|
| **Passwords** | Stored as SHA-256 hashes. Plaintext passwords typed into the Users sheet are silently upgraded on first login. An empty password is rejected before any lookup, so a row with a blank password cell can never sign in. |
| **Session Tokens** | Expiring tokens stored in GAS `CacheService`. Validated and refreshed on every authenticated request. |
| **APP_SECRET** | Required Script Property. Signs session tokens and derives the biometric encryption keystream. Hard-fails with a clear error if missing — no insecure fallback. |
| **Login Rate Limiting** | After **5 failed attempts** within **15 minutes**, the username is temporarily locked. Counter in `CacheService`; clears on success or after 15 minutes. |
| **Password Reset Rate Limiting** | Reset code requests capped at **3 per hour** per username. Codes expire after 15 minutes and are single-use. Response is identical whether or not the username exists — no enumeration. |
| **Face Data Encryption** | Descriptors encrypted at rest (HMAC-SHA256 stream cipher keyed on `APP_SECRET`, random IV per save, string-based HMAC input for Apps Script compatibility). See Section 4.10. |
| **Role Scoping** | Distinct roles (Admin, HR, Security-1/2, Standby, Employee), each restricted to exactly the screens they need. Security roles get Attendance Entry + History only — no Dashboard, Reports, or Admin access. |
| **Self-Deletion Guard** | An Admin cannot delete their own account. Enforced by both `deleteUser()` backend and the Admin Panel UI (Delete button disabled when editing your own account). |
| **Concurrency** | `LockService.waitLock(15000)` prevents duplicate punch rows during heavy shift-change windows. |
| **Audit Trail** | All admin edits, password resets, and correction requests are immutably appended to `Audit_Log` with timestamp, actor, before, and after values. |
| **Backups** | Full-sheet nightly backup to a separate Drive spreadsheet, 30-day retention, independent of any manual process. See Section 4.10. |
| **Storage isolation** | Every browser storage name is prefixed for this app, so it can neither read nor clear a sibling CALGAS app's session, cache, or offline data on the shared `calgas.github.io` origin. |

---

## 14. Version History

### v13.1 — Montserrat & Offline Safety (October 2026, current)

| Area | Change |
|---|---|
| **Typography** | Whole app in **Montserrat** — UI, headings, buttons, inputs, kiosk name label, printed payslip (embedded in the saved PDF), and the offline page. Self-hosted variable font (`fonts/montserrat-latin-var.woff2`, 34 KB, weights 100–900, SIL OFL) replaces the Roboto / Work Sans request to Google Fonts. A size-matched local fallback keeps the layout from jumping while it loads. |
| **Digits** | Tabular (fixed-width) digits app-wide, so the clock, counters and table columns don't shift as numbers change. |
| **Narrow phones** | Mobile header and dashboard stat cards tightened below 420px for Montserrat's wider letterforms; card titles no longer run under the Edit / Mark Resolved badge. |
| **Kiosk offline** | A recognised punch that can't reach the server is now saved to the offline queue (with recognition time) instead of being lost behind a toast. |
| **Wi-Fi without internet** | Manual entry queues the punch when the request fails before reaching the server, instead of showing "Your punch has been saved offline" while saving nothing. Timeouts are reported as "not confirmed" and never queued twice. |
| **Service Worker** | Out of the API path — no longer answers API calls on the page's behalf. Precaches the font. |
| **Messages** | Unreachable server now reads "Can't reach the server. Check the internet connection and try again." for login, password reset and every other call. |
| **Presentation** | Headings and body in Montserrat (repo copy first, Google Fonts when opened on its own); code stays monospace. |

### v13 — CALGAS Edition (October 2026)

The app, previously built and run for another plant, set up for CALGAS Capacitors on a fresh sheet.

| Area | Change |
|---|---|
| **Branding** | App name **CALGAS Workforce** (short name **CALGAS HR**). CALGAS logos served from the repo's own `logo/` folder (same files as Stock Management). Loader, login, sidebar, mobile header, offline page, CSV/Excel names and titles all CALGAS. |
| **Theme** | Accent colour is CALGAS indigo `#3d3f94` from the logo in light mode, with a lighter tint of the same hue (`#7478e0`) in dark mode for contrast. Accent tints are driven by a single `--accent-rgb` token. |
| **Payslip letterhead** | CALGAS wordmark, "CALGAS Mobility Pvt. Ltd.", and the Navsari plant address in the header; CALGAS footer. |
| **Professional Tax** | Gujarat slab (nil up to ₹12,000, ₹200 above) in `professionalTax()` (`Code.gs`), the Excel Salary Report formula, and the `List_of_Empl` PT formula. |
| **Mail** | New `sendMail_()` helper sends as `noreply@calgas.in` via `GmailApp`, falling back to the owner's address if the alias isn't verified. `checkMailSender()` added to confirm the sender and trigger authorization. |
| **Hosting** | `calgas/Workforce` → `calgas.github.io/Workforce/`. `manifest.json` uses relative paths (no hard-coded folder). Old placeholder screenshots removed. |
| **Shared origin** | Storage renamed for this app alone — cache prefix `calgas-workforce-`, IndexedDB `CalgasWorkforceDB` / `CalgasWorkforceSWMeta`, `localStorage` `wf_session` / `wf_theme`. Cache cleanup touches only this app's prefix. |
| **Backend** | `GOOGLE_API_URL` reset to the placeholder so this build can never talk to an earlier deployment's sheet. Month-settings tab is `HS` (still accepts `H/S`). Backup folder "CALGAS Workforce Backups". Script-cache keys renamed. Empty passwords rejected at login. |
| **Sheet template** | New `CALGAS_Workforce_Template.xlsx`: 8 clean tabs, Setup tab, open-ended J1/K1 formulas, Gujarat PT formula, 2026–2027 Sundays and national holidays, starter Admin and `REGULAR` shift rows. |

### Earlier versions (condensed)
- **v12 (July 2026):** Excel Salary Report brought in line with the payslip (sandwich rule and VPF applied, ADV. made additive after Net). Editable ADV. + Final Payable + print-inclusion toggles on the payslip, with ADV. carried into the print window and a manual Print button instead of an auto-print timer. History unified into one table with server-side employee filtering (fixed a hang from an undeclared `totalOTMins`). Security-1/Security-2 exposed in the user dropdown. Visual refresh: Work Sans headings, micro-interactions, light mode by default.
- **v11 (July 2026):** Face data encryption at rest, nightly Drive backups, self-service password reset, category-wise dashboard breakdown, visual leave calendar preview, Year-to-Date payslip summary, slide-down Help Guide. Post-testing fixes: HMAC byte-array bug, Help Guide content bleed, light-mode contrast, landscape rotation, the 701–767px layout gap, and font loading.
- **v10 (June 2026):** Mobile header spacer measured from the real header height. Change-history comment tags removed in favour of forward-looking comments.
- **v9 (May 2026):** 13 bug fixes, 6 performance upgrades, 7 structural changes — single-pass leave marking, real CO totals, self-deletion guard, hard-fail on missing `APP_SECRET`, sandwich-rule safe defaults, `leaveType` in offline payloads, single-employee status refresh, `Empl_ID` link for Employee logins, and the `app-version` meta tag.

---

## 15. Known Limitations

1. **Google Apps Script 6-minute execution limit:** Very large full-year CSV exports may time out. Filter by month.
2. **iOS Safari:** Background Sync API is unsupported. Requires the app to be open when the device reconnects to flush the offline punch queue.
3. **Face API on low-end mobile:** WebGL/GPU rendering depends on the device chipset. The `inputSize: 160` setting significantly mitigates this.
4. **No PWA screenshots:** The manifest has no `screenshots` entries, so Chrome shows its basic install prompt. Add 540×720 (narrow) and 1280×720 (wide) CALGAS screenshots to the repo and list them in `manifest.json` for the richer prompt.
5. **Face data encryption is a lightweight stream cipher, not AES:** Apps Script has no native AES implementation. The current HMAC-SHA256 keystream cipher protects against casual spreadsheet access but is not a substitute for a hardware-backed KMS if a stronger guarantee is ever required.
6. **Mail quota:** `GmailApp` sends under the script owner's daily Gmail quota.
7. **Landscape mode falls back to the desktop layout:** Functional, but any viewport wider than 767px uses the same collapsed-sidebar layout built for tablets/desktops rather than a purpose-built landscape phone layout.
8. **Data!J1/K1 must be pasted by hand:** Excel can't carry the Google-only `ARRAYFORMULA`/`LET` formulas, so they are applied once after importing the template.
9. **Attendance rules live in the sheet formulas:** The lunch deduction, overtime blocks and Sunday weekly off are in the J1/K1 formulas and the payroll code. A different CALGAS rule means changing the formula (and, for the weekly off, the backend's Sunday checks).
10. **HS day overrides apply to the payslip only:** `getEmpDashData()` reads No. of Days from the `HS` tab; the Excel Salary Report always uses calendar days. They agree while `HS` holds calendar days (the template default).
11. **Gujarat Labour Welfare Fund** (half-yearly employee/employer contribution) is not calculated.
12. **Excel exports keep Excel's default font.** A font doesn't travel inside an `.xlsx`, so Montserrat would become a substitute on any PC without it installed; on-screen, print and PDF output all use Montserrat.
13. **Characters outside Latin + ₹** (e.g. Gujarati or Devanagari names) fall back to the device's system font, since the font file is subset to keep it small.

---

## 16. Troubleshooting

**"Developer Error: Add API URL"**
`GOOGLE_API_URL` in `index.html` is still the placeholder. Paste the Web App URL (Section 7.1) — in `sw.js` too.

**`"APP_SECRET script property is not configured"`**
Add `APP_SECRET` to Apps Script → Project Settings → Script Properties before anyone can log in — this also blocks any face-data save/read, since the same secret drives biometric encryption. See Section 7.2.

**`"Too many failed login attempts. Please try again in 15 minutes."`**
Five consecutive failed logins triggered the rate limiter. Wait 15 minutes — the counter clears automatically.

**`"Session Expired"` immediately after login / API Permission Error**
Re-deploy the Apps Script as a **New Deployment** and confirm *Who has access* is set to **Anyone**. Update `GOOGLE_API_URL` in both `index.html` and `sw.js`.

**Dates show the wrong month, or attendance lands on the wrong day**
The Google Sheet's locale isn't India. File → Settings → Locale: India, Time zone: IST (Section 8, step 2).

**Hours / OT columns are empty**
The J1/K1 `ARRAYFORMULA`s haven't been pasted into the `Data` tab (Section 6).

**App won't install / Service Worker fails to install**
A file listed in `sw.js` `SHELL_ASSETS` is missing on GitHub Pages — usually a logo or the font. Upload the whole `fonts/` and `logo/` folders, keeping the file names exactly (including `CALGAS CAPACITORS-logo-768x240.jpg` with its spaces).

**Text shows in Arial instead of Montserrat**
`fonts/montserrat-latin-var.woff2` isn't on GitHub Pages, or an old cached build is running. Upload the `fonts/` folder, then use the refresh button or Ctrl+Shift+R.

**"Can't reach the server. Check the internet connection and try again."**
The request never left the device — no internet, Wi-Fi without internet, or a wrong `GOOGLE_API_URL` (check the constant's name and value in both `index.html` and `sw.js`). Punches made at that moment are queued and sync automatically; logins and password resets need the connection back.

**"Not confirmed — the server took too long"**
Google didn't answer within 30 seconds. The punch may or may not have been recorded — check the employee's status badge (manual entry) or History (kiosk) before punching again.

**Camera frozen on "Starting Camera..."**
The site must be served over **HTTPS**. Confirm the GitHub Pages URL uses `https://`.

**Face enrollment / re-enrollment fails with an HMAC or "Save Failed" error**
Confirm `Code.gs` has been redeployed as a **New Deployment** after updating, and that `GOOGLE_API_URL` in `index.html`/`sw.js` points to the new deployment URL.

**Leave day count mismatch between preview and actual days marked**
The preview counts working days excluding Sundays. The server additionally skips public holidays — the preview label states this.

**Employee payslip shows another employee's data (Employee-role login)**
Column E (`Empl_ID`) in the Users sheet is blank or wrong for this login. The user must log out and back in after fixing it.

**Kiosk stays on "Hold still..." and never punches**
The 2-of-3 frame confirmation is working correctly — the same face must appear in 2 consecutive scan frames. Re-enroll via the Admin panel if it persists.

**Monthly Excel CO column showing 0**
SOT bonus credits are only written when the employee punches OUT and elapsed time is ≥ 12 hours. Manually entered punch-out times don't trigger the flag retroactively.

**Password reset email never arrives**
Check column C (Email) is populated for that username. Run `checkMailSender()` to see which address mail is sent from, and check the owner account's Sent folder and daily Gmail quota.

**Password reset email comes from a personal/owner address instead of noreply@calgas.in**
The alias isn't verified on the script owner's account. Add it under Gmail → Settings → Accounts → Send mail as, then re-run `checkMailSender()` (Section 7.4).

**Signing in/out here signs me out of Stock Management (or the reverse)**
Shouldn't happen — this app keeps its session under `wf_session`. If it does, an old build of either app is cached; use the refresh button or hard-refresh.

**Service Worker not updating after a new deploy**
Confirm `<meta name="app-version" content="YYYYMMDD">` in `index.html` was updated (a same-day revision suffix like `20261007b` is fine). If not using the meta tag, bump `CACHE_DATE_FALLBACK` in `sw.js`.

**App won't rotate to landscape / rotating shows no navigation at all**
If rotation itself doesn't work, the app must be **reinstalled** (not just refreshed) after a manifest change — an installed PWA doesn't re-read `manifest.json` on a normal reload.

---

*CALGAS Workforce — CALGAS Capacitors (CALGAS Mobility Pvt. Ltd.), Navsari, Gujarat.*
*v13.1 — October 2026*
