# CampusFlow — Teacher Management System

A full-stack teacher management system for colleges, with department directories, secure staff accounts, self-service teacher sign-up, academic sessions, conflict-checked weekly timetables, audited attendance, server-computed reports with CSV and PDF exports, leave workflows, in-app notifications, self-service profiles, and an automated integration test suite.

**Current status: all eight phases complete.** The workspace, database, API baseline, responsive UI, authentication, administration, academic modules, attendance, reporting, leave, notifications, and profiles are working. Phase 8 added Vitest/Supertest integration tests running against an isolated in-memory MongoDB, a hardening pass on transport headers and production configuration, and the security review and deployment checklist in [`SECURITY.md`](SECURITY.md). A follow-up to Phase 8 added public teacher sign-up (always a teacher account, active immediately, rate-limited) and a controlled password-reset command for account recovery. The latest follow-up adds PDF/CSV routine upload: an administrator uploads the 5-day, 8-period routine, the server fills in unassigned teachers with a balanced subject-aware plan, the preview shows every conflict before anything is written, publishing replaces the session timetable, and every teacher receives a notification plus an optional daily class digest.

---

## 1. Current Phase (Phase 8 — complete)

### What works now

- npm workspace with separate `client/` and `server/` packages
- Express 5 API with a MongoDB/Mongoose connection
- Environment validation at startup (`server/src/config/env.js`)
- Security baseline:
  - Helmet HTTP headers
  - CORS allowlist driven by `CLIENT_ORIGIN`
  - Origin allowlist on state-changing requests (CSRF defense-in-depth)
  - Global API rate limiting plus a stricter sign-in limiter
  - Request IDs on every response
  - Consistent JSON success and error envelopes
  - No secrets in source code; `.env` files are git-ignored
- Health endpoints with real MongoDB ping checks
- React 19 + Vite 8 + Tailwind CSS 4 client
- Responsive dashboard shell:
  - Collapsible desktop sidebar
  - Mobile drawer with backdrop and Escape-to-close
  - Sticky top bar, theme toggle, notification preview, account menu
- Light/dark theme with persisted preference and no flash on load
- Reusable UI components, toasts, empty states, skeletons, and validated forms
- Design-system page with Recharts chart primitives (clearly marked sample data), admin-only
- Authentication:
  - Sign-in page with React Hook Form + Zod validation, show/hide password, Caps Lock hint, and accessible error alerts that include the request ID
  - Passwords hashed with bcrypt (12 rounds); plaintext passwords are never stored
  - JWT session token signed HS256 with issuer/audience, delivered in an HttpOnly, SameSite=Lax cookie (`Secure` in production, 8-hour default lifetime)
  - Sessions re-verified against the database on every protected request; deactivated accounts are rejected immediately
  - Generic "Invalid email or password." responses and timing-equalized credential checks
  - Sign-in rate limit: 10 failed attempts per 10 minutes per IP; successful sign-ins do not count
  - Sign-out clears the cookie; auth responses are marked `Cache-Control: no-store`
  - Role-based access control enforced on the server (`admin`, `teacher`), with `RequireAuth` and `RequireRole` on the client
  - Role-filtered navigation, a dedicated 403 page, and an account menu showing the current session
  - Self-service teacher sign-up at `/register`: name, email, optional phone, and a bcrypt-hashed password
  - The registration endpoint always creates a `teacher` with `approvalStatus: "pending"`; a `role`, `isActive`, or `admin` value sent by the client is ignored
  - A new sign-up gets no session cookie and cannot sign in (login returns `403`) until an administrator approves it
  - Administrators review requests at `/teachers/approvals`: approve to open sign-in and add the teacher to the directory, or reject with an optional note shown to the applicant
  - Pending and rejected accounts are excluded from the staff directory, dashboard counts, department statistics, head-teacher selection, and assignment lookups on the server
  - Sign-up is limited to five accounts per hour per IP by default (`REGISTER_RATE_LIMIT_*`), and each new account notifies active administrators with a link to the approvals queue
  - Controlled provisioning CLIs (`seed:admin`, `seed:user`) and a password-reset CLI (`reset:password`) for account recovery; administrators are never created from a public form
- Administration (Phase 3):
  - Admin overview with live counts for departments and teachers, account status, recent teacher accounts, and a department distribution chart
  - Department directory: create, edit, assign an active teacher as head, search, filter, paginate, activate, and deactivate
  - Teacher directory: create, edit, search, filter by department and status, paginate, reset passwords, activate, and deactivate
  - Email uniqueness, employee ID uniqueness, department scoping, and password rules enforced on the server
  - Deactivating a teacher blocks sign-in and revokes existing sessions; resetting a password does the same
  - Department deactivation is refused with `409` while active teachers remain
  - Admin-only routes and API endpoints for departments, teachers, and the overview
  - Demo seeding for five departments and six teachers with `npm run seed:demo`
- Academics (Phase 4):
  - Academic sessions with an exclusive current session (partial unique index); activating one deactivates the others
  - Semesters 1–8 per session, constrained to the session's start and end dates
  - Sections per department and semester with capacity, and classrooms with capacity and building
  - Subject catalogue per department and semester with unique, case-insensitive codes and credit values
  - Teaching assignments (session + section + subject, unique) validated against active sessions, subjects, sections, and teachers, with matching department and semester
  - Weekly timetable grid by section, teacher, or subject, built from active teaching assignments
  - Server-side clash checks for the section, teacher, and classroom; classroom capacity must cover the section
  - Timetable slots copy their session, semester, teacher, and period times from the assignment; periods 1–8 run from 09:00 to 16:20 (P1 09:00–09:50 through P8 15:30–16:20), Monday to Saturday
  - Current period, next period, and teaching day computed on the server from `COLLEGE_TIMEZONE`
  - Teacher timetable reads are forced to the signed-in teacher's account, even if another `teacher` query is supplied
  - Assignment edits and deactivations are refused while timetable slots reference them; subject, section, and classroom deactivation is refused while current-session slots use them
  - Demo seeding now includes academic data: one active session, two semesters, six sections, six classrooms, eleven subjects, fourteen assignments, and fourteen slots
- Attendance (Phase 5):
  - One attendance session per timetable slot and date; a unique index prevents duplicate sessions
  - Administrators open a check-in for a period, issue a new code, close the session, or mark attendance manually
  - A four-digit check-in code appears on the assigned teacher's own timetable (grid and today's classes) while the period window is open (15 minutes before the start until the end); the teacher copies it into the ongoing period and checks in
  - The server opens the check-in automatically the first time the teacher loads their day inside the window; the code is visible only in that teacher's authenticated response and is never sent to administrators
  - Codes are generated with `crypto.randomInt`, stored as a bcrypt hash for verification plus a display copy inside the session document, allow five attempts, can be refreshed after 60 seconds, and are capped at five issues per session
  - A code stays valid until the period's window closes; the copies are deleted when the check-in succeeds, the session closes, or attendance is marked manually
  - Server-side window validation uses `COLLEGE_TIMEZONE` and the college clock, never the browser clock; a check-in at least 10 minutes after the start is recorded as late
  - Per-user rate limits on code issuance and verification attempts; a locked code requires manual marking
  - Manual marking accepts present, late, absent, or excused; corrections require a reason and keep the prior status, new status, author, and timestamp in the record
  - The attendance day view lists every scheduled period for a date with summary counts, filters by department, teacher, and state, and shows code status (never the code itself) to administrators
  - Teachers see today's check-in panel and their own paginated attendance history; the day data, current period, and lockouts are all computed on the server
  - No mail server or SMTP configuration is needed; the check-in code is intentionally shown in the app, and the security trade-off is recorded in `SECURITY.md`
- Reports and audit (Phase 6), admin only:
  - Attendance report computed on the server from timetable entries and attendance records for a date range, with optional department and teacher filters
  - Default range is the last 30 days ending on the college date; ranges longer than 366 days, future end dates, or `from` after `to` return `422`
  - Buckets for scheduled, recorded, present, late, absent, excused, and unmarked; completion rate is recorded ÷ scheduled and attendance rate is (present + late) ÷ recorded, both rounded and `0` when the denominator is zero
  - Daily trend, per-department totals, per-teacher totals, and a daily breakdown limited to teaching days
  - Audit trail derived from each record's mark time and author plus every correction (previous status, new status, author, reason, timestamp), newest first, paginated and filterable to marks or corrections
  - CSV exports use a UTF-8 BOM, RFC 4180 quoting, CRLF line endings, and a formula-injection guard that prefixes cells starting with `=`, `+`, `-`, or `@`
  - PDF exports use PDFKit in landscape A4 with a branded header, metric cards, repeating table headers, page numbers, and "Page X of Y" footers
  - Downloads are streamed from the API with `Content-Disposition` filenames and `Cache-Control: no-store`; filenames are derived from the range, never from user input
  - The Reports page is admin-only on both the client route guard and the API; teachers receive `403` for report, audit, and export endpoints
- Leave, notifications, and profiles (Phase 7):
  - Teachers submit leave requests with a type, first and last day, and a reason; the server rejects past dates, ranges longer than 60 days, and requests that overlap another pending or approved request with `409`
  - Durations are calculated inclusively on the server and stored on the request, so reports and lists never depend on the browser
  - Teachers can withdraw a pending or approved request until its last day; ended or already-cancelled requests cannot be withdrawn
  - Administrators list every request with status, type, teacher, and date-range filters plus a summary of pending, approved, and upcoming approved days
  - Approving a request records the reviewer and timestamp; rejecting requires a note of at least three characters, and the note is shared with the teacher
  - Every decision is applied with a conditional update, so two administrators cannot decide the same request twice
  - Leave creation is teacher-only and review is admin-only on the API; a teacher receives `403` from the review endpoint
  - In-app notifications are created when leave is requested, approved, rejected, or withdrawn, and when an administrator corrects an attendance record
  - Notification endpoints are scoped to the signed-in user; read and read-all actions cannot touch another account's inbox
  - The notification bell shows an unread count, polls it every 60 seconds while the tab is visible, opens a recent-items panel, and links to a paginated inbox with All and Unread tabs
  - The profile page shows the signed-in user's identity fields, department, last sign-in, and current session dates
  - Users can update only their own name and phone number; email, role, department, employee ID, designation, and qualification stay administrator-managed, and an attempted email or role change is ignored
  - Password changes verify the current password, reject reusing it, hash the new password with bcrypt, invalidate every other session through `sessionVersion`, and re-issue the cookie so the current tab stays signed in
  - Password changes are rate-limited per user (5 per 15 minutes)
- Quality and hardening (Phase 8):
  - Vitest + Supertest integration suite: 80 tests across seven files covering transport hardening, authentication and roles, teacher sign-up and the approval workflow, leave, notifications, profiles, and routine upload/preview/publish/digests
  - Tests run against a disposable in-memory MongoDB per file; no fixture touches the development database and no test depends on another
  - Fixtures sign in through the real `/api/auth/login` endpoint, so cookie, session-version, and role checks are exercised end to end
  - `/api` responses are marked `Cache-Control: no-store`, ETags are disabled, and URL-encoded bodies are no longer parsed because the API is JSON-only
  - Startup configuration refuses non-HTTPS `CLIENT_ORIGIN` values and malformed origins when `NODE_ENV=production`
  - `SECURITY.md` documents the threat model, the controls in place, residual risks, and a production deployment checklist
- Routine upload and daily digests (latest follow-up):
  - Admins open **Routine upload** from the timetable page and upload a PDF, CSV, or TXT routine (5 MB limit), or paste text; a downloadable grid template is offered on the same page
  - The routine is read row-and-column wise: a header with period columns, one row per day, and one cell per class written as `Subject · Section · Teacher · Room`; a section header can be declared once for the whole grid, and one-class-per-line files are still accepted
  - Every parsed slot is previewed as a periods × days grid per section before publishing, with a status (ready, review, blocked), the auto-filled teacher or room, and a reason for each problem
  - Blank or unrecognized teachers are filled with a balanced, subject-aware plan: an existing teaching assignment for that section and subject wins, then a teacher who already teaches the subject, then the same department, with the lightest current load chosen; a teacher is never double-booked
  - Blank rooms are filled from active classrooms that are free at that slot and large enough for the section, preferring the smallest suitable room
  - Conflicts for the section, teacher, or classroom block publishing with a row-level message; the preview also shows how many existing slots the upload will replace
  - Publishing is admin-only, re-validates every id and clash on the server, creates the missing teaching assignments, replaces the active session's entries in one step, and notifies each teacher with a link to their timetable
  - Each teacher can receive one daily digest of that day's classes through `POST /api/timetable/digest` or the scheduled job; the digest is deduplicated per teacher and date, and skipped on Sundays
  - Both notification kinds appear in the bell, the inbox, and the notification counts with the same read/unread behaviour as the existing notifications

### Security posture at a glance

- Public sign-up creates pending `teacher` accounts only; administrators are created only through `npm run seed:admin`, and a role sent to the registration endpoint is ignored
- Sign-up gets no session and no access until an administrator approves it; approval, rejection, and the pending queue are admin-only on the server
- Sign-up is rate-limited per IP (five accounts per hour by default) and notifies active administrators
- Passwords are stored as bcrypt hashes, and check-in codes are stored as bcrypt hashes plus a display copy that is returned only to the assigned teacher's own day view and removed once the check-in is used
- Every protected route authenticates the cookie, reloads the user, checks `isActive` and `sessionVersion`, and then applies role checks on the server
- Rate limits cover the API globally, sign-in, registration, check-in code issuance/verification, and password changes
- See [`SECURITY.md`](SECURITY.md) for the full review and residual risks

### Not included

- Multi-factor authentication, SSO, and email verification flows
- Email-confirmation for sign-ups, and an in-app "forgot password" flow (recovery uses the `reset:password` command); approval is in-app, so administrators must sign in to review the queue
- Distributed rate-limit storage (the limiter uses in-memory state, which is per instance)
- Database-level exclusion constraints for overlapping leave (standalone MongoDB has no transactions; the service checks before insert)

Every page in the workspace requires an authenticated session, and every API route except health, sign-in, and teacher sign-up is scoped to the signed-in account or role. Teacher accounts can be requested from the sign-up page (then approved by an administrator), created directly by an administrator in the directory, or provisioned by the seed commands; administrators are created only with `npm run seed:admin`. Before exposing this build to the internet, work through the deployment checklist in [`SECURITY.md`](SECURITY.md).

---

## 2. Technology Stack

| Layer | Technology |
| --- | --- |
| Frontend | React 19, Vite 8 |
| Styling | Tailwind CSS 4 |
| Routing | React Router 7 |
| Icons | Lucide React |
| Charts | Recharts 3 |
| Forms | React Hook Form + Zod |
| HTTP client | Axios |
| Backend | Node.js, Express 5 |
| Database | MongoDB with Mongoose 9 |
| Exports | CSV with RFC 4180 quoting; PDF reports with PDFKit |
| Security | Helmet, CORS + Origin allowlist, express-rate-limit, bcrypt, JWT in HttpOnly cookies |
| Config | dotenv + Zod validation |

Phase 8 tests use Vitest and Supertest against an in-memory MongoDB; see section 6 for the commands.

---

## 3. Project Structure

```text
teacher-management-system/
├── client/
│   ├── public/
│   │   └── favicon.svg
│   ├── src/
│   │   ├── components/
│   │   │   ├── academic/        # Session, semester, section, classroom, subject, assignment, and timetable modals and tabs
│   │   │   ├── admin/           # ActiveBadge, PersonCell, DepartmentChart, DepartmentFormModal, TeacherFormModal
│   │   │   ├── attendance/      # Day table, teacher check-in panel, history table, mark modal, summary cards
│   │   │   ├── auth/            # AuthShell (sign-in/sign-up layout), RequireAuth, RequireRole
│   │   │   ├── leave/           # Summary cards, table, request/review/details modals
│   │   │   ├── notifications/   # NotificationPanel dropdown
│   │   │   ├── profile/         # Details card, profile form, password form
│   │   │   ├── reports/         # Summary cards, trend chart, teacher/daily tables, audit trail
│   │   │   └── ui/              # Button, Card, Badge, Field, Modal, ConfirmDialog, Pagination, TableSkeleton, ...
│   │   ├── constants/           # academics.js, attendance.js, leave.js (types, statuses, dates), notifications.js (icons)
│   │   ├── context/             # ThemeProvider, ToastProvider, AuthProvider, NotificationProvider
│   │   ├── hooks/               # useHealth, useDebouncedValue, usePaginatedList, useOptions
│   │   ├── layouts/             # DashboardLayout, Sidebar, Topbar
│   │   ├── pages/
│   │   │   ├── admin/           # AdminDashboardPage
│   │   │   ├── DashboardPage.jsx
│   │   │   ├── AcademicSetupPage.jsx
│   │   │   ├── SubjectsPage.jsx
│   │   │   ├── TimetablePage.jsx
│   │   │   ├── AttendancePage.jsx
│   │   │   ├── LeavePage.jsx
│   │   │   ├── NotificationsPage.jsx
│   │   │   ├── ProfilePage.jsx
│   │   │   ├── ReportsPage.jsx
│   │   │   ├── DepartmentsPage.jsx
│   │   │   ├── TeachersPage.jsx
│   │   │   ├── StaffOverviewPage.jsx
│   │   │   ├── LoginPage.jsx
│   │   │   ├── RegisterPage.jsx
│   │   │   ├── DesignSystemPage.jsx
│   │   │   ├── ForbiddenPage.jsx
│   │   │   └── NotFoundPage.jsx
│   │   ├── services/            # Axios client, auth, health, admin, department, teacher, academic, subject, assignment, timetable, attendance, report, leave, notification, profile
│   │   ├── utils/               # cn(), formatting, roles, validation, form errors, password generator
│   │   ├── App.jsx
│   │   ├── main.jsx
│   │   ├── index.css
│   │   └── navigation.js
│   ├── .env.example
│   ├── index.html
│   ├── package.json
│   └── vite.config.js
├── server/
│   ├── src/
│   │   ├── config/              # env.js, db.js, periods.js
│   │   ├── controllers/         # auth, admin, department, teacher, academic, subject, assignment, timetable, attendance, report, leave, notification, profile
│   │   ├── middleware/          # requestId, rateLimiter, authenticate, authorize, verifyOrigin, notFound, errorHandler
│   │   ├── models/              # User, Department, AcademicSession, Semester, Section, Classroom, Subject, TeachingAssignment, TimetableEntry, AttendanceSession, AttendanceRecord, LeaveRequest, Notification
│   │   ├── routes/              # index, health, auth, admin, department, teacher, academic, subject, assignment, timetable, attendance, report, leave, notification, profile
│   │   ├── scripts/             # createUser.js, resetPassword.js, seedDemo.js
│   │   ├── services/            # auth, admin, department, teacher, academic, subject, assignment, timetable, attendance, report, export, leave, notification, profile
│   │   ├── utils/               # ApiError, sendResponse, logger, password, tokens, query, time
│   │   ├── validators/          # auth, common, department, teacher, academic, subject, assignment, timetable, attendance, report, leave, notification, profile
│   │   ├── app.js
│   │   └── server.js
│   ├── tests/                   # helpers.js plus API integration tests (Vitest + Supertest)
│   ├── .env.example
│   ├── package.json
│   └── vitest.config.js
├── .gitignore
├── package.json                 # npm workspaces + dev, seed, and test scripts
├── SECURITY.md                  # Threat model, controls, residual risks, deployment checklist
└── README.md
```

The `server/tests/` directory boots an in-memory MongoDB per test file, seeds users with real bcrypt hashes, and drives the Express app through Supertest.

---

## 4. Prerequisites (Windows)

1. **Node.js 20.19 or newer** — install the LTS build from <https://nodejs.org/>. Verify in Command Prompt:

   ```bat
   node -v
   npm -v
   ```

2. **MongoDB** — choose one:
   - **Local MongoDB Community Server** from <https://www.mongodb.com/try/download/community>. During setup, install it as a Windows service so it starts automatically. The default local URI is `mongodb://127.0.0.1:27017/teacher_management`.
   - **MongoDB Atlas** (cloud, no local install): create a free cluster, create a database user, allow your IP under *Network Access*, then copy the connection string (use `mongodb+srv://...`).
   - **Dev-only in-memory MongoDB** (no installation): set `USE_IN_MEMORY_DB=true` in `server/.env`. The first start downloads a MongoDB binary (approximately 780 MB) into a local cache. Data disappears when the API stops. Never use this in production.

No mail server is involved. Check-in codes are generated by the API during the period window and shown only on the signed-in teacher's own timetable and attendance page, so development needs no extra configuration.

Account seeding needs a persistent database. Set `USE_IN_MEMORY_DB=false` and point `MONGODB_URI` at local MongoDB or Atlas before creating accounts; the seed script refuses to run against the temporary in-memory database.

---

## 5. Installation (Windows Command Prompt)

Open **Command Prompt** (not PowerShell, unless you prefer it) and run:

```bat
cd C:\path\to\teacher-management-system
npm install
```

This installs the root, client, and server dependencies through npm workspaces.

Create the environment files:

```bat
copy server\.env.example server\.env
copy client\.env.example client\.env
```

Open `server\.env` in Notepad:

```bat
notepad server\.env
```

Configure one of the following:

**A. Local MongoDB or Atlas (recommended)**

```env
MONGODB_URI=mongodb://127.0.0.1:27017/teacher_management
USE_IN_MEMORY_DB=false
```

For Atlas, replace the URI with your `mongodb+srv://...` string and add the database name, for example:

```env
MONGODB_URI=mongodb+srv://USER:PASSWORD@cluster0.xxxxx.mongodb.net/teacher_management?retryWrites=true&w=majority
```

**B. Dev-only in-memory MongoDB (no MongoDB install)**

```env
USE_IN_MEMORY_DB=true
```

Keep `CLIENT_ORIGIN=http://localhost:5173` unless you change the Vite port.

**C. Set a session secret and create the first administrator**

The API refuses to start without `JWT_SECRET`. Generate a random value:

```bat
node -e "console.log(require('crypto').randomBytes(48).toString('base64url'))"
```

Paste the result into `server\.env` as `JWT_SECRET=...` (32–256 characters). Keep this value private and use a different one in production.

Anyone can request a teacher account at <http://localhost:5173/register>. The request is created as pending with no access and waits in **Teachers → Approvals** until an administrator approves or rejects it. Sign-up can never grant the admin role, so the first administrator still comes from the seed command. Add these lines to `server\.env` temporarily:

```env
ADMIN_NAME=Your Name
ADMIN_EMAIL=you@college.edu
ADMIN_PASSWORD=ChangeMe2026-Secure9
```

Then run from the project root:

```bat
npm run seed:admin
```

The script validates the input, refuses duplicate emails, hashes the password with bcrypt, and never prints it. Store the password in a password manager — it cannot be recovered. Delete the three `ADMIN_*` lines from `server\.env` afterwards.

To create an additional teacher account, set `SEED_NAME`, `SEED_EMAIL`, and `SEED_PASSWORD` in `server\.env` and run:

```bat
npm run seed:user -- --role=teacher
```

Passwords must be 10–72 characters and include at least one letter and one number. Remove the `SEED_*` values after use. The seed commands exit with an error instead of creating an account when the role, email, or password is invalid or the email already exists.

### Recovering an account password

If you lose an administrator password, or need to reset any account without using the admin UI, run the reset command from the project root. It reads the new password from `RESET_PASSWORD`, hashes it with bcrypt, and bumps the account's `sessionVersion`, so every existing session for that account is revoked:

```bat
set RESET_PASSWORD=ChangeMe2026-Recover9
npm run reset:password -- --email=admin@college.edu
```

In Git Bash:

```bash
RESET_PASSWORD='ChangeMe2026-Recover9' npm run reset:password -- --email=admin@college.edu
```

`RESET_EMAIL` works instead of `--email=`. The command refuses an unknown email, a weak password, and `USE_IN_MEMORY_DB=true`, never prints the password or hash, and leaves a deactivated account deactivated. Sign in with the new password, then change it from **Your profile**.

### Optional demo data

For a populated local environment, add a shared password for the demo accounts to `server\.env`:

```env
DEMO_PASSWORD=ChangeMe2026-Demo4
```

Then run:

```bat
npm run seed:demo
```

This creates five departments (Computer Science, Mathematics, Physics, Business Administration, English), six teacher accounts, and assigns two department heads. It also creates the 2026-27 academic session (activated only when no other session is active), two semesters, six sections, six classrooms, eleven subjects, fourteen teaching assignments, and fourteen timetable slots across Monday to Thursday. The script is idempotent: existing records and emails are skipped. Demo accounts share `DEMO_PASSWORD`; never reuse that password outside local development.

Reports read live attendance records, so a freshly seeded database shows the scheduled periods as unmarked until someone marks or checks in. Sign in as an administrator, mark a few periods from **Attendance**, and the Reports page, CSV, and PDF populate from those records. Corrections made from the attendance day view appear in the audit trail immediately.

Leave requests and notifications start empty. Sign in as a teacher, open **Leave**, and submit a request; then sign in as an administrator, open **Leave**, and approve or reject it. The decision creates a notification in the teacher's bell, and withdrawing the request notifies administrators again.

Set the college timezone used by the timetable clock in `server\.env`:

```env
COLLEGE_TIMEZONE=Asia/Kolkata
```

The server computes the current period, next period, teaching day, and attendance window from this IANA timezone, not from the browser clock.

### Routine uploads

An administrator uploads the weekly routine from **Timetable → Upload routine** (`/timetable/import`). The page accepts a PDF, CSV, or TXT file up to 5 MB, or pasted text. A generated grid template with the current session's real sections, subjects, and rooms is included in `client/public/templates/`; regenerate it from live data with:

```bat
npm run template:routine --workspace server
```

The routine is read as a **row-and-column grid**: a header names the period columns, each following row is a day, one cell per class:

```text
Section | A
Department | CS
Day | P1 | P2 | P3 | P4 | P5 | P6 | P7 | P8
Monday | CS101 · Ananya Rao · CS-101 | CS102 | ...
Tuesday | CS101 · Rohan Mehta | CS102 · Priya Nair · CS-102 | ...
```

Each cell reads `Subject · Section · Teacher · Room`, in that order, with `·`, `/`, or a spaced dash as the separator. `Section` may be declared once on a `Section | A` line (with an optional `Department | CS` line) before the header, so the cells only need the subject and, optionally, the teacher and room. `Day` accepts a name or `1–6`; period columns may be labelled `P1`–`P8`, `Period 1`–`Period 8`, or `1`–`8`. Names and codes match ignoring case, spacing, and honorifics, and blank teachers and rooms are filled during preview and marked **Auto**.

A flat one-class-per-line table is still accepted in the same order:

```text
Day | Period | Department | Section | Subject | Teacher | Room
Monday | 1 | Computer Science | A | CS101 | Ananya Rao | CS-101
```

The preview shows each section as a periods × days grid (switch sections with the tabs when the routine covers more than one), so you can check the whole week slot by slot before publishing.

Routine uploads are preview-only until an administrator publishes. The preview lists parsed slots, ready, review, and blocked counts, the number of existing slots that would be replaced, and a reason for every problem. A teacher is never double-booked, and a room is never assigned twice in the same slot; both cases block publishing for that row. Publishing replaces all timetable slots in the chosen session, creates any missing teaching assignments, and sends one notification to each teacher in the routine. The confirmation dialog states how many slots will be replaced.

Daily digests are sent by the API's scheduled job, which runs at `DAILY_DIGEST_CRON` (default `0 7 * * 1-6`, college timezone) and skips Sundays. An administrator can also trigger today's digest from the routine page. Each teacher gets at most one digest per day; rerunning the job or the endpoint does not create duplicates.

### Attendance check-in codes

The API opens a check-in automatically the first time a teacher loads their timetable or attendance page inside a period window (15 minutes before the start until the period ends). A four-digit code appears on that teacher's timetable and on the ongoing-period panel; the teacher copies it into the panel and the server verifies the bcrypt hash before recording the period.

The code is valid until the window closes, can be reissued after the 60-second cooldown, and never appears in an administrator's response, an API log, or a notification. Because the code is displayed in the app rather than delivered out of band, it proves control of the signed-in session and presence at the period, not possession of a mailbox; this trade-off and its compensating controls are recorded in [`SECURITY.md`](SECURITY.md).

The client `.env` can stay as-is:

```env
VITE_API_URL=/api
```

---

## 6. Running the App

From the project root:

```bat
npm run dev
```

This starts both processes with `concurrently`:

- API: <http://localhost:5000>
- Web app: <http://localhost:5173>

Open <http://localhost:5173>. You are redirected to the sign-in page, which links to a teacher sign-up form at `/register`. A sign-up shows a confirmation screen and waits for approval; it cannot sign in yet and does not appear in the directory. Administrators land on the admin overview with live department, teacher, academic, and today's-attendance counts, a pending-approvals card when requests are waiting, a distribution chart, quick actions (including **Reports**, **Leave approvals**, and **Teacher approvals**), and recent teacher accounts. The Reports page adds range and department/teacher filters, summary tiles, a daily trend, per-department and per-teacher tables, a daily breakdown, and an audit trail with CSV and PDF downloads. The **Leave** page shows every request with summary cards, filters, and review actions; the notification bell opens recent items and links to a paginated inbox. The account menu opens **Your profile**, where name, phone, password, and session details are managed. Teachers see a workspace with API status, database state, uptime, today's classes from the live timetable, their attendance status, and their current session details, plus their own leave requests and notifications. Use the account menu in the top bar to sign out.

If the Vite server starts on a different port because 5173 is busy (for example 5174), add that origin to `CLIENT_ORIGIN` in `server\.env` and restart the API; otherwise same-origin GETs work but POST requests such as sign-out are rejected with `403`.

### Signing in and roles

| Role | Access in Phase 8 |
| --- | --- | --- |
| `admin` | Overview, attendance day view and manual marking, reports and exports, audit trail, leave approvals, teacher approvals, notifications, profile, timetable, teachers, departments, subjects and assignments, academic setup, and the design-system page |
| `teacher` | Personal workspace, read-only own timetable, in-app check-in code for today's periods, personal attendance history, own leave requests and withdrawals, notifications, and profile; admin pages — including **Reports** and leave review — return the 403 screen |

The server checks the session cookie and the user's role on every protected API request. Client-side route guards exist for a better experience, not as the security boundary. Teacher navigation includes the timetable, attendance, and leave; every timetable and attendance read is filtered to the signed-in teacher on the server, leave lists and withdrawals are scoped to the owner, notifications are scoped to the signed-in account, and only the assigned teacher can check in to a period.

Protected routes behave as follows:

- A guest visiting `/dashboard` is redirected to `/login`; after signing in, the app returns to the requested page unless the role cannot open it, in which case the dashboard opens instead.
- A signed-in user visiting `/login` or `/register` is redirected to `/dashboard`.
- A sign-up succeeds without signing in: the confirmation screen shows the email and the three approval steps, and signing in before approval returns a clear `403` message.
- An administrator approves or rejects the request at `/teachers/approvals`; approval opens sign-in and adds the teacher to the directory.
- A signed-out or expired session returns `401` from the API and the app returns to `/login`.
- A teacher opening an admin-only route sees the **403 Restricted** page, and the admin-only approval endpoints return `403` to the API.

The first start with `USE_IN_MEMORY_DB=true` waits for the MongoDB binary download, so the API may take several minutes to become ready. Later starts are fast.

### Run each side separately

```bat
npm run dev:server
```

```bat
npm run dev:client
```

### Other commands

| Command | Purpose |
| --- | --- |
| `npm run dev` | Run API and client together |
| `npm run dev:server` | API only, with nodemon |
| `npm run dev:client` | Vite dev server only |
| `npm run seed:admin` | Create the first administrator from `ADMIN_*` variables in `server\.env` |
| `npm run seed:user -- --role=teacher` | Create a teacher from `SEED_*` variables in `server\.env` |
| `npm run reset:password -- --email=user@college.edu` | Reset an account password from `RESET_PASSWORD` and revoke its sessions |
| `npm run seed:demo` | Create demo departments and teacher accounts from `DEMO_PASSWORD` |
| `npm run template:routine --workspace server` | Regenerate the grid CSV and PDF routine templates from live sections, subjects, and rooms into `client/public/templates/` |
| `npm run build` | Production build of the client into `client/dist` (Vite may warn that the main chunk is over 500 kB because form validation libraries are shared; Recharts stays in the lazy design-system chunk) |
| `npm run preview` | Preview the production client build |
| `npm test` | Run the API integration suite once (Vitest + Supertest, in-memory MongoDB) |
| `npm run test:watch --workspace server` | Re-run the suite on file changes while developing |
| `npm start` | Start the API without nodemon (requires a real `MONGODB_URI`; in-memory mode is blocked in production) |

### Running the test suite

```bat
npm test
```

The suite starts a disposable in-memory MongoDB for each test file, creates users with real bcrypt hashes, signs in through the real API, and then exercises the endpoints with Supertest. It never reads `server/.env` values for the database, never writes to your development database, and needs no mail server. The first run may take a minute while the MongoDB test binary is prepared; later runs take about 20 seconds.

Test files:

| File | Covers |
| --- | --- |
| `server/tests/api-hardening.test.js` | Health output, `no-store`, Helmet headers, ETag suppression, request IDs, 404s, malformed JSON, form-encoded rejection, CORS and origin rules |
| `server/tests/auth.test.js` | Sign-in cookie flags, generic failures, session reload, deactivation, tampering, session-version revocation, sign-out, role authorization, teacher sign-up (pending account, no cookie, blocked sign-in, forced role, duplicate `409`, validation, admin notification) |
| `server/tests/approvals.test.js` | Pending queue and directory exclusion, approval opening sign-in and notification, rejection with a note, teacher `403`, invalid decisions, unknown ids, and repeat decisions (`409`) |
| `server/tests/leave.test.js` | Day counting, past and long ranges, overlap conflicts, teacher-only creation, admin-only review, rejection notes, approval notifications, withdrawal rules, role-scoped listings |
| `server/tests/notifications.test.js` | Admin fan-out, unread counts, single and bulk read, cross-account isolation, pagination |
| `server/tests/profile.test.js` | Profile fields, name/phone updates, ignored email/role writes, password verification, session rotation, per-user throttling |
| `server/tests/routineImport.test.js` | CSV and PDF parsing, balanced subject-aware auto-assignment, unknown-teacher warnings, publish replacement and teacher notifications, digest creation and deduplication, teacher `403` on routine endpoints |

---

## 7. Verify the API

With the API running:

```bat
curl.exe http://localhost:5000/api/health
```

Expected response (values vary):

```json
{
  "success": true,
  "message": "API is running",
  "data": {
    "status": "ok",
    "environment": "development",
    "uptimeSeconds": 12,
    "timestamp": "2026-10-28T12:00:00.000Z",
    "database": { "state": "connected", "reachable": true }
  }
}
```

Other endpoints:

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/` | API pointer |
| GET | `/api` | Service metadata |
| GET | `/api/health` | Liveness + database status |
| GET | `/api/health/live` | Liveness only |
| GET | `/api/health/ready` | Readiness; returns `503` if MongoDB is unreachable |
| POST | `/api/auth/login` | Validate credentials, set the session cookie |
| POST | `/api/auth/register` | Create a pending teacher account and notify administrators (`201`, no session cookie); rate-limited per IP; any `role` in the body is ignored |
| POST | `/api/auth/logout` | Clear the session cookie |
| GET | `/api/auth/me` | Return the current user and session details (requires a session) |
| GET | `/api/admin/overview` | Admin dashboard totals, department breakdown, recent teachers |
| GET | `/api/departments` | List departments (`search`, `status`, `page`, `limit`) |
| POST | `/api/departments` | Create a department |
| GET | `/api/departments/:id` | Read one department |
| PATCH | `/api/departments/:id` | Update a department or its head |
| PATCH | `/api/departments/:id/status` | Activate or deactivate a department |
| GET | `/api/teachers` | List approved teachers (`search`, `status`, `department`, `page`, `limit`) |
| GET | `/api/teachers/approvals` | Admin-only queue of pending sign-ups (`search`, `page`, `limit`), oldest first |
| PATCH | `/api/teachers/:id/approval` | Admin decides a pending sign-up: `{ "decision": "approve" \| "reject", "note": "optional" }`; `409` if already decided |
| POST | `/api/teachers` | Create a teacher account and profile |
| GET | `/api/teachers/:id` | Read one teacher |
| PATCH | `/api/teachers/:id` | Update a teacher; a new password revokes sessions |
| PATCH | `/api/teachers/:id/status` | Activate or deactivate a teacher account |
| GET | `/api/academic/summary` | Academic counts and the active session |
| GET | `/api/academic/sessions` | List academic sessions (`search`, `status`, `page`, `limit`) |
| POST | `/api/academic/sessions` | Create a session; `isActive: true` activates it exclusively |
| GET | `/api/academic/sessions/:id` | Read one session with its semester count |
| PATCH | `/api/academic/sessions/:id` | Update session dates or name, validated against its semesters |
| PATCH | `/api/academic/sessions/:id/status` | Activate (deactivating the others) or deactivate a session |
| GET | `/api/academic/semesters` | List semesters (`academicSession`, `number`, `status`, pagination) |
| POST | `/api/academic/semesters` | Create a semester inside its session's dates |
| GET/PATCH | `/api/academic/semesters/:id` | Read or update one semester |
| PATCH | `/api/academic/semesters/:id/status` | Activate or deactivate a semester |
| GET | `/api/academic/sections` | List sections (`department`, `semester`, `status`, pagination) |
| POST | `/api/academic/sections` | Create a section for a department and semester |
| GET/PATCH | `/api/academic/sections/:id` | Read or update one section |
| PATCH | `/api/academic/sections/:id/status` | Activate or deactivate a section; blocked while current-session slots use it |
| GET | `/api/academic/classrooms` | List classrooms (`search`, `status`, pagination) |
| POST | `/api/academic/classrooms` | Create a classroom with capacity and building |
| GET/PATCH | `/api/academic/classrooms/:id` | Read or update one classroom |
| PATCH | `/api/academic/classrooms/:id/status` | Activate or deactivate a classroom; blocked while current-session slots use it |
| GET | `/api/subjects` | List subjects (`search`, `department`, `semester`, `status`, pagination) with active-session assignment counts |
| POST | `/api/subjects` | Create a subject with a unique code |
| GET/PATCH | `/api/subjects/:id` | Read or update one subject |
| PATCH | `/api/subjects/:id/status` | Activate or deactivate a subject; blocked while assignments or slots use it |
| GET | `/api/assignments` | List assignments (`search`, `academicSession`, `department`, `subject`, `section`, `teacher`, `status`, pagination) |
| POST | `/api/assignments` | Assign a teacher to a subject for a section in the active session |
| GET/PATCH | `/api/assignments/:id` | Read or update one assignment; edits are blocked while slots exist |
| PATCH | `/api/assignments/:id/status` | Activate or deactivate an assignment; deactivation is blocked while slots exist |
| GET | `/api/timetable` | Weekly slots for the active session (`section`, `teacher`, `subject`, `semester`, `day`, pagination); teachers are scoped to themselves |
| GET | `/api/timetable/current` | College time, current and next period, active-session status, and the teacher's classes today |
| POST | `/api/timetable` | Create a slot (`assignment`, `classroom`, `dayOfWeek`, `periodNumber`) with clash checks |
| POST | `/api/timetable/routine/preview` | Admin-only routine preview from a multipart `file` (PDF/CSV/TXT) or `text`; returns parsed rows, auto-assignments, issues, and replacement counts without writing |
| POST | `/api/timetable/routine/publish` | Admin-only; validate and replace the session timetable with the previewed rows, then notify every teacher in the routine |
| POST | `/api/timetable/digest` | Admin-only; send (or resend safely) the daily class digest for a date, defaulting to the college date |
| PATCH | `/api/timetable/:id` | Move or reassign a slot with the same validation |
| DELETE | `/api/timetable/:id` | Remove a slot; the teaching assignment is kept |
| GET | `/api/attendance/day` | Scheduled periods for a date (`date`, `department`, `teacher`, `status`) with sessions, availability, and summary; teachers see only their own periods |
| GET | `/api/attendance/today` | The signed-in teacher's own periods, availability, summary, and the four-digit code for each period currently inside its check-in window |
| GET | `/api/attendance/sessions` | Attendance sessions (`from`, `to`, `teacher`, `department`, `status`, `mode`, pagination); teachers are scoped to themselves |
| GET | `/api/attendance/sessions/:id` | Read one attendance session; access is scoped to the teacher or an administrator |
| POST | `/api/attendance/sessions` | Open a check-in for a timetable slot and date (assigned teacher only; window enforced; rate-limited); the code is returned only to that teacher |
| POST | `/api/attendance/sessions/:id/resend` | Issue a new code after the 60-second cooldown, up to five sends (rate-limited) |
| POST | `/api/attendance/sessions/:id/check-in` | Verify the four-digit code and record present or late; locks after five attempts (rate-limited) |
| PATCH | `/api/attendance/sessions/:id/close` | Admin closes an open session; attendance stays unmarked |
| POST | `/api/attendance/records` | Admin marks or corrects attendance (`timetableEntry`, `date`, `status`, `note`); corrections require a reason |
| GET | `/api/reports/attendance` | Attendance report for a date range (`from`, `to`, `department`, `teacher`) with summary, teachers, departments, and daily rows |
| GET | `/api/reports/attendance/export.csv` | CSV download of the same report |
| GET | `/api/reports/attendance/export.pdf` | PDF download of the same report |
| GET | `/api/reports/audit` | Paginated audit trail (`from`, `to`, `department`, `teacher`, `action`, `page`, `limit`) with mark/correction counts |
| GET | `/api/reports/audit/export.csv` | CSV download of every audit event in the range |
| GET | `/api/reports/audit/export.pdf` | PDF download of every audit event in the range |
| GET | `/api/leaves` | List leave requests (`status`, `type`, `teacher`, `page`, `limit`); teachers see only their own |
| GET | `/api/leaves/summary` | Pending, approved, cancelled, and upcoming approved days; teachers see their own totals |
| POST | `/api/leaves` | Teacher submits a leave request (`type`, `startDate`, `endDate`, `reason`); overlap and date rules are enforced |
| PATCH | `/api/leaves/:id/cancel` | Teacher withdraws a pending or approved request before its last day |
| PATCH | `/api/leaves/:id/review` | Admin approves or rejects a pending request (`status`, `note` required when rejecting) |
| GET | `/api/notifications` | Paginated notifications for the signed-in user (`unread`, `page`, `limit`) with the unread count |
| GET | `/api/notifications/unread-count` | Unread count for the signed-in user, used by the bell poller |
| PATCH | `/api/notifications/:id/read` | Mark one notification read; other users' notifications return `404` |
| PATCH | `/api/notifications/read-all` | Mark every notification for the signed-in user read |
| GET | `/api/profile` | Signed-in user's profile with department and session details |
| PATCH | `/api/profile` | Update the signed-in user's name and phone; identity fields are ignored |
| POST | `/api/profile/change-password` | Verify the current password, rotate it, invalidate other sessions, and re-issue the cookie (rate-limited) |

Report endpoints default to the last 30 days ending on the college date, reject invalid ranges with `422`, and export with `Content-Type: text/csv; charset=utf-8` or `application/pdf`, an attachment filename derived from the range, and `Cache-Control: no-store`. CSV files start with a UTF-8 BOM so Excel opens them correctly. PDFs are landscape A4 with repeating headers and page numbers.

The admin, department, teacher, academic, subject, assignment, and report routes require an authenticated `admin` session. Routine preview, publish, and digest endpoints are admin-only for the same reason: only administrators may rewrite the session timetable. A teacher receives `403` from those endpoints even if the request is made outside the browser. For `GET /api/timetable` and `GET /api/timetable/current`, any signed-in user may read, but a teacher's query is forced to their own `teacher` id; timetable writes stay admin-only. Attendance reads follow the same rule: administrators see the whole college, teachers see only their own sessions. Only the assigned teacher can request a code or check in, and only administrators can mark, correct, or close. Leave creation and withdrawal are teacher-only and review is admin-only; list and summary responses are scoped on the server, so a teacher cannot read another teacher's requests. Notification reads and writes are scoped to the signed-in account, and the profile API only accepts name and phone changes. List responses use `{ success, message, data: [...], meta: { page, limit, total, totalPages } }`; validation failures return `422` with field-level details and conflicts return `409`.

A request to an unknown route returns `404` with a consistent JSON error envelope and a `requestId`. Malformed JSON returns `400`, an oversized payload returns `413`, and more than 300 requests per 15 minutes from one IP returns `429` (health checks are exempt). Sign-in has its own stricter limit: 10 failed attempts per 10 minutes per IP, after which the API returns `429` with a clear message. Successful sign-ins do not count toward that limit. Registration is limited separately to five accounts per hour per IP (`REGISTER_RATE_LIMIT_WINDOW_MINUTES`, `REGISTER_RATE_LIMIT_MAX`), so a public deployment cannot be flooded with accounts.

### Verify authentication from the command line

Windows builds of `curl` can keep the session cookie in a file:

```bat
curl.exe -i -X POST http://localhost:5000/api/auth/login -H "Content-Type: application/json" -d "{\"email\":\"you@college.edu\",\"password\":\"YOUR_PASSWORD\"}" -c cookies.txt
```

A successful response is `200`, sets an `HttpOnly; SameSite=Lax` cookie, and returns the user with a session expiry. The JWT is never included in the JSON body. Then:

```bat
curl.exe http://localhost:5000/api/auth/me -b cookies.txt
curl.exe -i -X POST http://localhost:5000/api/auth/logout -b cookies.txt
```

Expected results: `/me` returns `200` with the user and session while the cookie is valid; logout returns `200` and clears the cookie; `/me` without a valid cookie returns `401` with `Sign in to continue.` A tampered or expired token also returns `401`. Wrong credentials return `401` with the deliberately generic `Invalid email or password.` Deactivated accounts return `403`.

Browser requests with a state-changing method must come from an origin listed in `CLIENT_ORIGIN`; untrusted origins receive `403`. Requests with no `Origin` header (curl, server-to-server) are allowed, which is why these examples work without extra headers. Delete `cookies.txt` after testing.

To confirm the error envelope from Command Prompt:

```bat
curl.exe -i -X POST http://localhost:5000/api/health -H "Content-Type: application/json" -d "{bad json"
```

The response is `400` with `{"success":false,"message":"Invalid JSON in the request body.","requestId":"..."}`.

---

## 8. Environment Variables

### `server/.env`

| Variable | Default | Notes |
| --- | --- | --- |
| `NODE_ENV` | `development` | `development`, `test`, or `production` |
| `PORT` | `5000` | Must match the Vite proxy target if changed |
| `MONGODB_URI` | none | Required unless `USE_IN_MEMORY_DB=true`; must start with `mongodb://` or `mongodb+srv://` |
| `CLIENT_ORIGIN` | `http://localhost:5173` | Comma-separated trusted origins |
| `TRUST_PROXY` | `false` | Set `true` only behind a trusted reverse proxy |
| `RATE_LIMIT_WINDOW_MINUTES` | `15` | Global API rate-limit window |
| `RATE_LIMIT_MAX` | `300` | Requests per window per IP |
| `USE_IN_MEMORY_DB` | `false` | Development only; blocked when `NODE_ENV=production` and incompatible with account seeding |
| `JWT_SECRET` | none | **Required.** 32–256 characters. Never commit the real value; changing it invalidates every session |
| `SESSION_TTL_HOURS` | `8` | Session lifetime, 1–720 hours |
| `SESSION_COOKIE_NAME` | `campusflow_session` | Name of the HttpOnly cookie |
| `AUTH_RATE_LIMIT_WINDOW_MINUTES` | `10` | Window for counting failed sign-ins |
| `AUTH_RATE_LIMIT_MAX` | `10` | Failed sign-ins per IP per window before `429` |
| `REGISTER_RATE_LIMIT_WINDOW_MINUTES` | `60` | Window for counting teacher sign-ups |
| `REGISTER_RATE_LIMIT_MAX` | `5` | Sign-ups per IP per window before `429` |
| `ADMIN_NAME`, `ADMIN_EMAIL`, `ADMIN_PASSWORD` | none | Read only by `npm run seed:admin`; remove after creating the admin |
| `SEED_NAME`, `SEED_EMAIL`, `SEED_PASSWORD` | none | Read only by `npm run seed:user`; remove after creating an account |
| `RESET_EMAIL`, `RESET_PASSWORD` | none | Read only by `npm run reset:password`; pass them for that one command and do not save them |
| `DEMO_PASSWORD` | none | Read only by `npm run seed:demo`; shared by local demo accounts. Remove after seeding |
| `COLLEGE_TIMEZONE` | `Asia/Kolkata` | IANA timezone used to compute the current period, teaching day, and attendance window server-side |
| `ATTENDANCE_CHECKIN_EARLY_MINUTES` | `15` | How early before the period start the check-in window opens |
| `ATTENDANCE_LATE_AFTER_MINUTES` | `10` | Minutes after the start at which a check-in is recorded as late |
| `ATTENDANCE_OTP_MAX_ATTEMPTS` | `5` | Incorrect attempts before the code locks |
| `ATTENDANCE_OTP_RESEND_SECONDS` | `60` | Cooldown before a new code can be issued |
| `ATTENDANCE_OTP_MAX_SENDS` | `5` | Maximum codes issued per session |
| `OTP_SEND_RATE_LIMIT_WINDOW_MINUTES`, `OTP_SEND_RATE_LIMIT_MAX` | `15`, `10` | Per-user code-issue limit |
| `OTP_VERIFY_RATE_LIMIT_WINDOW_MINUTES`, `OTP_VERIFY_RATE_LIMIT_MAX` | `15`, `20` | Per-user verification limit |
| `DAILY_DIGEST_ENABLED` | `true` | Starts the daily class digest job with the API; set `false` to run digests manually only |
| `DAILY_DIGEST_CRON` | `0 7 * * 1-6` | Five-field cron for the digest job, evaluated in `COLLEGE_TIMEZONE`; the default runs Monday to Saturday at 07:00 |

A code stays valid until its period's check-in window closes; there is no separate lifetime setting. No mail configuration exists: the code is shown only on the signed-in teacher's timetable and attendance page.

### `client/.env`

| Variable | Default | Notes |
| --- | --- | --- |
| `VITE_API_URL` | `/api` | `/api` uses the Vite dev proxy; set a full URL in production |

Never commit `.env` files. Only `.env.example` files belong in version control. The seed variables are commented out in `.env.example`; the running API never reads them. Passwords exist only as bcrypt hashes in MongoDB, and the session cookie is `HttpOnly`, `SameSite=Lax`, and `Secure` in production.

---

## 9. Troubleshooting

| Symptom | Fix |
| --- | --- |
| `[config] Invalid environment configuration` | Re-copy `server\.env.example` to `server\.env` and check that values are not blank or malformed. |
| `CLIENT_ORIGIN must use https:// in production` | Production refuses plain-HTTP origins so the session cookie stays protected. Serve the client over HTTPS and update `CLIENT_ORIGIN`. |
| `npm test` cannot start MongoDB | The first run downloads the test binary. Allow the download, or point `MONGOMS_SYSTEM_BINARY` at an existing `mongod` executable. |
| Tests pass locally but fail in CI | Run `npm test` from the repository root so the workspace resolves; the API listens on no fixed port during tests, so no port conflicts are expected. |
| `JWT_SECRET` validation error | Generate a 32–256 character secret (section 5) and paste it into `server\.env`; restart the API. |
| `Account was not created` | Fill `ADMIN_*` (for `seed:admin`) or `SEED_*` (for `seed:user`) in `server\.env`. Passwords need 10–72 characters with at least one letter and one number. |
| `An account with ... already exists` | Sign in with that account or use a different email; the seed and sign-up paths never overwrite an existing user. |
| Sign-up returns `429 Too many accounts created from this network` | The per-IP registration limit (default five per hour) is reached. Wait for the window, or raise `REGISTER_RATE_LIMIT_MAX` locally. |
| Sign-in says the account is waiting for administrator approval | Expected until reviewed. Sign in as an administrator, open **Teachers → Approvals**, and approve or reject the request. |
| An approved teacher signs in but the dashboard count is lower than expected | Only `approvalStatus: "approved"` accounts count. The API backfills accounts created before the approval workflow once at startup with `approvalStatus: "approved"`; check the server log for `Marked N existing account(s) as approved`. |
| `The password was not reset` | Pass a password of 10–72 characters with a letter and a number in `RESET_PASSWORD`, and an existing address in `--email=` or `RESET_EMAIL`. |
| Locked out of the only admin account | Run `npm run reset:password -- --email=admin@college.edu` with `RESET_PASSWORD` set, then sign in with the new password. |
| Seed fails with an in-memory database message | Set `USE_IN_MEMORY_DB=false` and use local MongoDB or Atlas; temporary data disappears when the script exits. |
| Signed in but an admin page shows 403 | That page is admin-only. Sign in with an account whose role is `admin`. |
| `429 Too many sign-in attempts` | Wait `AUTH_RATE_LIMIT_WINDOW_MINUTES` (default 10) or restart the API to clear the in-memory counter. |
| `Unable to connect to MongoDB` | Start the MongoDB Windows service, or fix `MONGODB_URI`. For Atlas, check the database user and IP allowlist. For a quick start, set `USE_IN_MEMORY_DB=true`. |
| In-memory DB download fails or is too slow | Install MongoDB Community locally (or use Atlas) and set `USE_IN_MEMORY_DB=false`. Corporate proxies may block the binary download. |
| `Port 5000 is already in use` | Stop the other process, or change `PORT` in `server\.env` **and** the Vite proxy target in `client\vite.config.js`. |
| Browser console CORS error | `CLIENT_ORIGIN` must exactly match the browser origin, including scheme and port. |
| Sign-out or any POST shows "This origin is not allowed to access the API" | Vite is probably running on a port other than 5173. Add the actual origin to `CLIENT_ORIGIN` (for example `http://localhost:5173,http://localhost:5174`) and restart the API. |
| Report percentages are 0% or every period is unmarked | No attendance has been recorded in the selected range. Mark periods from **Attendance**, widen the range, or clear the teacher and department filters. |
| Teacher sees 403 on Reports | Reports, audit, and exports are administrator-only by design. Sign in with an `admin` account. |
| Leave request returns an overlap error | A pending or approved request already covers one of the selected days. Withdraw it or pick different dates. |
| Rejection fails with "Enter a short note" | Rejecting requires a note of at least three characters; approving does not. |
| No Request leave button | Only signed-in teachers can request leave. Administrators review requests from the same Leave page. |
| Notification bell count lags behind | The bell polls every 60 seconds while the tab is visible; open the panel and press Refresh or reload the page. |
| Password change signs out another device | Expected: changing the password invalidates every other session and re-issues the current tab's cookie. |
| Overview shows "The API is not reachable" | Start the API (`npm run dev:server`) and confirm <http://localhost:5000/api/health> works. |
| `429 Too many requests` | Wait for the rate-limit window, or raise `RATE_LIMIT_MAX` locally. |
| No check-in code appears on the timetable | The code exists only while the period window is open (15 minutes before the start until the end) and only for the assigned teacher. Reload the timetable; if the window has closed, an administrator must mark the period. |
| "The check-in window has passed" | The period has ended. Mark the period manually as an administrator, or correct the record with a reason. |
| The teacher sees no check-in form | Check-in is available only while the window is open, for the teacher assigned to that slot. Administrators can mark the period manually or open a check-in. |
| The code is locked after five attempts | An administrator must mark or correct that period manually; the teacher cannot retry. |
| "Manual marking opens when the period starts" | Manual marking is intentionally unavailable before the scheduled start; wait for the period or ask an administrator to open a check-in. |
| Redirected to sign-in after every reload | Allow cookies for `localhost`; confirm `CLIENT_ORIGIN` matches the browser origin and the Vite proxy target matches the API `PORT`. |
| Blank page after a dependency change | Stop both servers, run `npm install` again, then `npm run dev`. |
| npm engine warning | Use Node 20.19+ (`node -v`). Node 22 LTS is recommended. |

---

## 10. Roadmap

| Phase | Scope | Status |
| --- | --- | --- |
| 1 | Setup, database, API baseline, responsive UI foundation | **Complete** |
| 2 | Authentication, bcrypt, JWT (HttpOnly cookies), roles, protected routes, admin seed | **Complete** |
| 3 | Admin dashboard, teacher management, departments | **Complete** |
| 4 | Subjects, teaching assignments, academic setup, timetable | **Complete** |
| 5 | Staff attendance sessions, in-app check-in codes, manual attendance and corrections | **Complete** |
| 6 | Attendance reports, percentages, CSV/PDF export, audit trail | **Complete** |
| 7 | Leave management, notifications, profile pages | **Complete** |
| 8 | Integration tests, security review, deployment hardening, documentation | **Complete** |

---

## 11. Phase 8 Verification Checklist

- [x] `npm test` runs 69 integration tests across five files and all pass against a disposable in-memory MongoDB
- [x] Each test file starts and stops its own database; no test depends on execution order or on the development database
- [x] Fixtures create users with real bcrypt hashes and sign in through `POST /api/auth/login`
- [x] Every `/api` response, including 404s, carries `Cache-Control: no-store`, and no ETag is emitted
- [x] Malformed JSON returns `400`; a form-encoded sign-in attempt is rejected with `422`
- [x] Requests from an untrusted `Origin` are rejected with `403`; requests without an `Origin` still reach validation
- [x] Sign-in sets an HttpOnly, SameSite=Lax cookie and never returns the token in the JSON body
- [x] A tampered cookie, a bumped `sessionVersion`, a deactivated account, and a signed-out cookie all fail closed (`401`/`403`)
- [x] Teachers receive `403` on admin-only endpoints; a role changed in the database takes effect without re-issuing the token
- [x] Leave tests cover the inclusive day count, past dates, the 60-day cap, overlap `409`, teacher-only creation, admin-only review, rejection notes, approval notifications, withdrawal limits, and role-scoped listings
- [x] Notification tests prove admin fan-out, unread counts, single and bulk reads, cross-account `404`, and pagination
- [x] Profile tests prove that only name and phone change, email and role writes are ignored, and a password change invalidates old cookies while the new cookie works
- [x] The per-user password-change limiter returns `429` on the sixth failed attempt
- [x] `NODE_ENV=production` refuses a non-HTTPS or malformed `CLIENT_ORIGIN`
- [x] `npm run build` still succeeds after the Phase 8 client copy changes
- [x] `SECURITY.md` documents the threat model, implemented controls, residual risks, and a deployment checklist
- [x] `POST /api/auth/register` creates a pending teacher, returns `201` with `requiresApproval: true`, sets no session cookie, and never returns the password or token
- [x] A pending account cannot sign in (`403` with an approval message) and has no access to any protected endpoint
- [x] A `role: "admin"` value submitted to sign-up is ignored; the stored account stays a teacher and receives `403` on admin endpoints
- [x] A duplicate email, including different casing and surrounding whitespace, returns `409`; invalid name, email, or password returns `422` with field errors
- [x] Each new account notifies active administrators (`account_pending`) and leaves deactivated admins untouched
- [x] `/register` validates name, email, phone, matching passwords, and password strength in the browser, shows a confirmation screen, and maps server field errors back to the form
- [x] `GET /api/teachers/approvals` and `PATCH /api/teachers/:id/approval` are admin-only; teachers receive `403`
- [x] Approving opens sign-in, adds the teacher to the directory, and sends an `account_approved` notification; rejecting keeps sign-in blocked and sends the note with `account_rejected`
- [x] A decided request cannot be decided again (`409`); an unknown id returns `404`; an invalid decision returns `422`
- [x] Pending accounts are excluded from the teacher directory, overview counts (`pendingApproval` is reported separately), department active counts, head-teacher selection, and assignment lookups
- [x] Accounts created before the workflow are backfilled to `approved` once at API startup, so existing staff keep working
- [x] `npm run reset:password` resets a password, revokes existing sessions, refuses weak passwords and unknown emails, and never prints the secret
- [x] Browser QA: sign-up shows the confirmation screen and no auto sign-in; pre-approval sign-in shows the blocked message; the admin dashboard shows the pending card and banner; the queue approves the request; the new teacher then signs in and opens the teacher workspace

---

## 12. Phase 7 Verification Checklist

- [x] A teacher can submit a leave request with a type, dates, and reason; the server computes the inclusive day count
- [x] A past start date returns `422`; a range longer than 60 days returns `422`
- [x] An overlapping pending or approved request returns `409` with a `startDate` field error
- [x] Administrators cannot create leave requests; teachers cannot review them (`403`)
- [x] Rejection without a note of at least three characters returns `422` with a `note` field error
- [x] Approving records the reviewer and timestamp; the teacher's bell receives the decision and note
- [x] Withdrawing a pending or approved request is allowed until its last day and notifies administrators
- [x] Ended requests cannot be withdrawn; another teacher's request cannot be cancelled
- [x] The leave summary reconciles with the list: pending, approved, cancelled, and upcoming approved days
- [x] The admin Leave page filters by status, type, and teacher and paginates correctly
- [x] The teacher Leave page shows only the signed-in teacher's requests and a Request leave action
- [x] The notification bell shows the unread count, opens recent items, and links to the full inbox
- [x] Marking one notification read decrements the count; Mark all read clears it
- [x] A user cannot read, mark, or count another account's notifications
- [x] The Notifications page switches between All and Unread and paginates
- [x] The profile page shows identity, department, last sign-in, and session dates
- [x] Name and phone updates persist; attempted email or role changes are ignored
- [x] Changing the password verifies the current value, rejects reusing it, and signs out other sessions while keeping the current tab signed in
- [x] Attendance corrections generate a notification for the affected teacher
- [x] Leave, notifications, and profile pages work in light and dark themes and on narrow screens
- [x] `npm run build` succeeds after the Phase 7 client modules

---

## 13. Phase 6 Verification Checklist

- [x] Report totals reconcile with the underlying attendance records across the summary, teacher, department, and daily rows
- [x] Completion rate is recorded ÷ scheduled and attendance rate is (present + late) ÷ recorded; both are `0` when the denominator is empty
- [x] The default range is the last 30 days ending on the college date; `from` after `to`, a future end date, or a range over 366 days returns `422`
- [x] Department and teacher filters constrain every section of the report consistently
- [x] Unmarked scheduled periods are counted and shown separately instead of being dropped
- [x] The audit trail lists every mark and correction newest first with actor, method, note, previous status, new status, and timestamp
- [x] The audit action filter returns only corrections when selected
- [x] Audit counts in the header match the filter and pagination metadata
- [x] CSV exports start with a UTF-8 BOM, quote commas/quotes/newlines per RFC 4180, use CRLF, and prefix formula-leading cells
- [x] PDF exports contain the title, range, session, metric cards, full table headers, and "Page X of Y" footers
- [x] Downloads set `Content-Type`, `Content-Disposition`, and `Cache-Control: no-store`, and use a filename derived from the range
- [x] Anonymous report requests return `401`; teacher requests return `403` for the report, audit, and both export formats
- [x] A teacher opening `/reports` directly sees the 403 screen, and Reports is absent from teacher navigation
- [x] The Reports page renders in light and dark themes; wide tables scroll horizontally on narrow screens
- [x] `npm run build` succeeds after the Phase 6 client modules

---

## 14. Phase 5 Verification Checklist

- [x] One session per timetable slot and date; duplicate requests return `409`
- [x] Only the assigned teacher can request a code or check in; other teachers and guests are refused
- [x] Check-in is allowed only inside the server-computed window, including the timezone and academic session
- [x] A check-in after the late threshold is recorded as late; before it, present
- [x] Codes are four digits, stored only as bcrypt hashes, and returned only to the assigned teacher for today's in-window periods
- [x] The code is never sent to administrators, written to logs, or included in notifications
- [x] A code stays valid until its period's window closes and locks after five incorrect attempts
- [x] Issuing a new code respects the 60-second cooldown and the five-code limit
- [x] Code issuance and verification endpoints are rate-limited per user
- [x] Administrators can open a check-in, issue a new code, close a session, mark manually, and correct a record
- [x] Corrections require a reason and preserve the previous status, new status, author, and timestamp
- [x] The attendance day view lists every scheduled period and filters by department, teacher, and state
- [x] Teachers see only their own periods, summary, and history
- [x] Teachers receive `403` from closing sessions, manual marking, and correction endpoints
- [x] Admin manual marking opens when the period starts; earlier rows show the window opening time
- [x] The admin day view and teacher check-in flow work in light and dark themes
- [x] The teacher timetable shows the code during the window and copies it to the clipboard on click
- [x] No mail server or SMTP settings are required in development or production
- [x] `npm run build` succeeds after the Phase 5 client modules

---

## 15. Phase 4 Verification Checklist

- [x] Only one academic session can be active; activating a session deactivates the others
- [x] Duplicate session names and duplicate semester numbers return `409`
- [x] Semester dates outside the session range return `422`
- [x] Sections are unique per department, semester, and name; classroom names are unique
- [x] Subject codes are unique case-insensitively; active-session assignment counts appear in the list
- [x] Teaching assignments require an active session, and an active subject, section, and teacher from the same department and semester
- [x] Duplicate assignments for a session, section, and subject return `409`
- [x] Duplicate teacher, section, or classroom slots return `409` with a human-readable clash message
- [x] A classroom smaller than the section returns `422`
- [x] Slots copy the assignment's session, teacher, semester, and period times; edits and deletes work
- [x] Assignment edits and deactivation are refused while timetable slots reference them
- [x] Subject, section, and classroom deactivation is refused while current-session slots use them
- [x] `GET /api/timetable` and `/api/timetable/current` return the signed-in teacher's own classes even when another `teacher` query is supplied
- [x] Teachers receive `403` from timetable writes, subject, assignment, and academic setup endpoints
- [x] Anonymous timetable requests return `401`
- [x] `/api/timetable/current` reports the college timezone, date, current period, next period, and active-session status
- [x] `npm run seed:demo` creates the active session, semesters, sections, classrooms, subjects, assignments, and slots idempotently
- [x] The admin timetable grid switches between section, teacher, and subject views, adds slots from empty cells, edits, and deletes
- [x] Conflict messages appear in the timetable modal with a request ID
- [x] The teacher workspace shows today's classes and the current/next period; `/timetable` is read-only and has no add controls
- [x] Teacher navigation shows the timetable but not academic administration pages
- [x] Desktop and dark theme were reviewed; `npm run build` succeeds

---

## 16. Phase 3 Verification Checklist

- [x] Roles are limited to `admin` and `teacher`; the student module and `/api/students` were removed
- [x] Admin overview reads live counts, the department breakdown, and recent teachers from MongoDB
- [x] Departments can be created, edited, searched, filtered, paginated, activated, and deactivated
- [x] Department deactivation is refused with `409` while active teachers remain
- [x] Department heads must be active teachers
- [x] Teachers can be created, edited, searched, filtered by department/status, and paginated
- [x] Duplicate emails and employee IDs return `409` with field-level details
- [x] Invalid teacher or department input returns `422` with field-level details
- [x] Deactivated teachers cannot sign in and existing sessions are revoked
- [x] Resetting a password revokes existing sessions
- [x] Teachers receive `403` from admin-only pages and API endpoints
- [x] Admin navigation shows Teachers and Departments; teacher navigation does not
- [x] `npm run seed:demo` creates five departments, six teachers, and two department heads idempotently
- [x] Desktop and dark theme were reviewed; `npm run build` succeeds

---

## 17. Phase 2 Verification Checklist

- [x] `JWT_SECRET` is required at startup and is not stored in source control
- [x] Admin created only through `npm run seed:admin`; no registration form and no admin option in the UI
- [x] Duplicate email is refused by the seed script with a clear error
- [x] Weak passwords are rejected before an account is created
- [x] `GET /api/auth/me` returns `401` without a cookie
- [x] Sign-in returns `200`, sets an `HttpOnly; SameSite=Lax` cookie, and never returns the JWT in the body
- [x] `/api/auth/me` returns the user and session with a valid cookie
- [x] A tampered cookie returns `401`
- [x] A deactivated account cannot sign in (`403`)
- [x] The 11th failed sign-in attempt returns `429` with a clear message
- [x] Untrusted origins are rejected with `403` on state-changing requests
- [x] Sign-out clears the cookie; `/me` then returns `401`
- [x] A guest visiting `/dashboard` is redirected to sign-in and returned afterwards
- [x] A signed-in user visiting `/login` is redirected to the overview
- [x] A teacher account opening an admin-only page sees the 403 screen
- [x] Navigation filters admin-only items by role
- [x] The account menu shows name, email, role, last sign-in, and signs out cleanly
- [x] The overview shows the current session details and expiry
- [x] Desktop (1440 px, dark theme) and narrow (≈638 px, light theme) layouts were reviewed
- [x] `npm run build` produces the production bundle

---

## 18. Phase 1 Verification Checklist (still passing)

- [x] `npm install` completes
- [x] `npm run dev` starts the API and client together
- [x] `npm run build` produces a production client bundle
- [x] API starts and connects to MongoDB
- [x] `GET /api/health` returns `200` with database reachability
- [x] Unknown API routes return a JSON `404`
- [x] Malformed JSON returns a JSON `400` envelope
- [x] Untrusted origins are rejected by CORS with a JSON `403` envelope
- [x] Overview page reads live health data
- [x] Sidebar collapses on desktop and becomes a drawer on mobile
- [x] Light/dark theme persists across reloads
- [x] Design-system page renders charts, forms, and feedback states
- [x] Unknown client routes render the 404 page
