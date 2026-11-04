# CampusFlow Security Review

This document records the security review completed in Phase 8: what the system
protects, which controls are implemented, how they are verified, what risk remains,
and what to check before the API and client face the internet.

It describes the code in this repository as of Phase 8. Re-read it whenever
authentication, authorization, or deployment topology changes.

---

## 1. Scope and deployment assumptions

CampusFlow is a **staff-only** application for colleges. It has two roles:

| Role | Purpose |
| --- | --- |
| `admin` | Manage departments, teacher accounts, academics, attendance, reports, leave approvals, and teacher sign-up approvals |
| `teacher` | Use a personal workspace: timetable with in-app check-in codes, own attendance history, own leave, notifications, and profile |

There is no student module. Teacher accounts can be requested through public
self-registration, created by an administrator, or provisioned by the controlled
seed commands. A self-registered account is created as `pending`: it gets no
session, cannot sign in, and is hidden from every staff query until an
administrator approves it. Administrators are never created from the sign-up
form or by choosing a role; they come only from `npm run seed:admin` or the
admin directory, and a `role` sent to the registration endpoint is ignored.

The review assumes:

- The API runs behind a TLS-terminating reverse proxy on a private network.
- MongoDB is reachable only from the API host.
- The client is served over HTTPS from the origin listed in `CLIENT_ORIGIN`.
- Only trusted operators can read `server/.env` and the process environment.

---

## 2. Assets and threat model

**Assets**

- Teacher and administrator accounts and their password hashes
- Session cookies and the JWT signing secret
- Attendance records and their audit trail
- Leave requests and administrative decisions
- Personal data: names, college email addresses, phone numbers, employee IDs
- The MongoDB connection string and other deployment secrets

**In scope**

- Credential stuffing and password guessing against `/api/auth/login`
- Session theft, replay, and fixation
- Privilege escalation from `teacher` to `admin`
- Cross-account access to attendance, leave, notifications, and reports
- CSRF on cookie-authenticated writes
- Attendance fraud: checking in outside the scheduled window, replaying or refreshing codes to bypass a lock, brute-forcing check-in codes
- Injection through request bodies, query strings, and route parameters
- Data exposure through caches, logs, error responses, and exports

**Out of scope for this review**

- Physical access to the host or database
- Compromise of the DNS/TLS provider
- Insider abuse by a legitimate administrator (administrators can already change teacher records by design)

**Trust boundaries**

1. Browser to API — untrusted; every request is authenticated and authorized on the server.
2. API to MongoDB — trusted network path, but all queries are built from validated input.
3. Administrator UI to API — same boundary as (1); admin role is enforced server-side.

---

## 3. Implemented controls

### 3.1 Authentication

- Passwords are hashed with **bcrypt, 12 rounds**; plaintext is never written to the database, logs, or responses.
- Sign-in compares the supplied password against the stored hash, or against a fixed dummy hash for unknown accounts, so response timing does not reveal whether an account exists.
- Failures always return `401` with **"Invalid email or password."**; deactivated accounts return `403`.
- Email addresses are normalized (trim + lowercase) before lookup by the Zod schema.
- The first administrator is created with `npm run seed:admin`; teachers can be created with `npm run seed:user`, from the admin directory, or requested through public sign-up and approved by an administrator.
- Changing a password requires the current password, rejects reuse, and increments `sessionVersion`.
- `npm run reset:password` requires shell access to the API host and the database, hashes the supplied password, and increments `sessionVersion` so every existing session is revoked. It never prints the password or hash.

### 3.1.1 Teacher self-registration

- `POST /api/auth/register` accepts only `name`, `email`, `phone`, and `password`. Unknown keys such as `role`, `isActive`, or `approvalStatus` are stripped by the Zod schema, and the service writes `role: 'teacher'`, `isActive: true`, and `approvalStatus: 'pending'` itself, so a client cannot escalate or approve itself.
- The password is validated (10–72 characters, letter and number) and hashed with bcrypt before insert; the plaintext is never logged or returned.
- A duplicate email returns `409` without revealing any other account details. A concurrent insert that trips the unique index is converted to the same `409`.
- Registration does **not** create a session: it returns `201` with `requiresApproval: true` and no `Set-Cookie` header.
- Sign-in and the `authenticate` middleware both refuse a non-approved account with `403`. Rejected accounts also fail closed, and rejection increments `sessionVersion`.
- Each new account notifies active administrators (`account_pending`) with the applicant's name, email, and a link to the queue; approvals and rejections notify the applicant (`account_approved`, `account_rejected`).
- `GET /api/teachers/approvals` and `PATCH /api/teachers/:id/approval` require the `admin` role. The decision is a server-side enum (`approve` or `reject`), the reviewer and timestamp are recorded, an optional note is capped at 300 characters, and a decided request cannot be decided again (`409`).
- Pending and rejected accounts are excluded from the teacher directory, overview counts, department active counts, head-teacher selection, and assignment lookups at the query level; the overview reports `pendingApproval` separately.
- Accounts that predate the workflow are backfilled to `approved` once at startup, so existing staff keep access.
- Registration is rate-limited per IP (five per hour by default) in addition to the global API limiter, which also limits how quickly an attacker can flood the approvals queue.

### 3.2 Sessions

- Sessions are **HS256 JWTs** signed with `JWT_SECRET`, with `issuer`, `audience`, `subject`, `iat`, and `exp`.
- The token lives in an **HttpOnly, SameSite=Lax cookie**, `Path=/`, `Secure` when `NODE_ENV=production`, 8-hour default lifetime (`SESSION_TTL_HOURS`).
- The token is **never returned in a JSON body** or stored in `localStorage`.
- `authenticate` reloads the user from MongoDB on **every protected request**, then checks:
  - the user still exists,
  - the account is active,
  - `sessionVersion` matches the token's `ver` claim.
- Deactivation and password changes therefore revoke existing sessions without a session table.
- Sign-out clears the cookie. API responses carry `Cache-Control: no-store` and ETags are disabled, so intermediaries cannot replay authenticated data.

### 3.3 Authorization

- Every protected route is mounted behind `authenticate`; role checks use `authorize(...roles)`.
- Roles are read from the **database on each request**, not from the JWT claim, so a role change or demotion takes effect immediately.
- Tenant-style scoping is applied in services, not only the client:
  - teachers can only read their own timetable and attendance,
  - leave lists and withdrawals are scoped to the owner,
  - notifications are scoped to the signed-in account,
  - reports and exports are admin-only.
- Client-side guards (`RequireAuth`, `RequireRole`) exist for user experience only; the server is the boundary.

### 3.4 CSRF and origins

- Session cookies are `SameSite=Lax`.
- State-changing requests pass through `verifyOrigin`, which rejects any `Origin` header that is not in the `CLIENT_ORIGIN` allowlist with `403`.
- CORS allows credentials only from the same allowlist, with an explicit method and header list.

### 3.5 Input validation and injection

- Every request body and query string is parsed by **Zod schemas**; unknown fields are stripped.
- Route parameters that must identify records are validated as 24-character ObjectIds before use (`mongoose.isValidObjectId` or `objectIdSchema`).
- Mongoose casting plus schema enums prevent operator injection; no request value is concatenated into a query.
- JSON bodies are limited to **1 MB**, and URL-encoded bodies are **not parsed** because the API is JSON-only.
- Invalid JSON returns `400`; schema failures return `422` with field-level messages.
- CSV exports neutralize formula injection (`=`, `+`, `-`, `@`) and quote per RFC 4180. PDF filenames are derived from validated dates, never from user text.
- React escapes rendered content; the app does not use `dangerouslySetInnerHTML`.
- Routine uploads are admin-only. The multipart handler accepts a single `file` field, keeps it in memory (never writes it to disk), caps it at **5 MB**, filters on the PDF/CSV/TXT MIME type and extension, and rejects PDFs over 20 pages. The preview body is limited to 200,000 characters of pasted text, and a routine is capped at 2,000 rows.
- Parsing never executes document content: PDF text is extracted with `unpdf`, and every cell is matched against records loaded from the database. Uploaded names cannot inject queries, and unknown values become row warnings rather than new records.
- Preview writes nothing. Publishing re-validates every id and every section, teacher, and classroom clash on the server, ignores client-supplied labels, and replaces the session timetable only after those checks pass. A teacher cannot call preview, publish, or digest.
- Digest and publish notifications are deduplicated by a per-user key with a partial unique index, so replaying a request cannot duplicate a teacher's inbox.

### 3.6 Attendance check-in codes

- The check-in window is computed **on the server** from `COLLEGE_TIMEZONE` and the timetable slot; the browser clock is never trusted.
- A code is issued only for the signed-in teacher's own periods, only on the current date, and only while that period's window is open. The admin day view never receives a code, and neither does another teacher.
- Codes are four digits generated with `crypto.randomInt` and are stored twice: a **bcrypt hash** used for verification (`otpHash`, excluded from serialization) and a display copy (`checkInCode`, `select: false`, removed by the `toJSON` transform) so the teacher can read it on their own timetable.
- A code stays valid until the period window closes, allows five attempts, can be refreshed after 60 seconds, and is capped at five issues per session plus the per-user send and verify rate limits. Both copies are deleted from the document when the check-in succeeds, the session closes, or attendance is marked manually.
- Successful check-ins are recorded with the server timestamp; late detection is server-side.
- Manual marks and corrections are audited with actor, reason, previous status, new status, and timestamp.

**Deliberate trade-off.** The previous design emailed the code, so a check-in proved possession of the teacher's mailbox. The code now appears in the authenticated teacher's own timetable. It therefore proves control of an active teacher session plus presence in the timetable during the window — not a second factor. Anyone holding a teacher's session could type the code they can already see; they could equally mark the period manually, which the same server window allows. The compensating controls are the teacher-only endpoint, server-side window validation, single use, the attempt lock, the resend caps, and the audit trail. If institutional policy requires proof of possession, reintroduce an out-of-band channel (email or authenticator) instead of the displayed code.

### 3.7 Rate limiting

| Scope | Default |
| --- | --- |
| All API routes (health exempt) | 300 requests / 15 min / IP |
| Sign-in | 10 failed attempts / 10 min / IP (successes do not count) |
| Teacher registration | 5 / 60 min / IP |
| Check-in code send (open/refresh) | 10 / 15 min / user |
| Check-in code verify | 20 / 15 min / user (successes do not count) |
| Password change | 5 / 15 min / user (successes do not count) |

Limiter responses return `429` with a request ID and no stack details.

### 3.8 Transport and headers

- Helmet sets a CSP, `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, HSTS, and other defaults on API responses.
- `x-powered-by` is disabled.
- The CORS allowlist is exact-match; there are no wildcards.
- `trust proxy` is enabled only when `TRUST_PROXY=true`, so client IPs come from the proxy only when that is explicitly configured.
- In production the process refuses to start if `CLIENT_ORIGIN` contains a non-HTTPS or malformed origin.

### 3.9 Configuration and secrets

- `server/src/config/env.js` validates every variable with Zod at startup and exits on invalid input.
- Production refuses in-memory MongoDB.
- Secrets (`JWT_SECRET`, `MONGODB_URI`, seed passwords) come only from the environment or `server/.env`; none are in source code, and `.env` files are git-ignored.
- `.env.example` contains placeholders only. Seed variables are read by CLI scripts and can be removed after use.

### 3.10 Error handling and logging

- Central error handling returns a consistent envelope: `{ success, message, errors?, requestId }`.
- In production, 5xx responses do not include stack traces or internal messages.
- `requestId` is attached to every response for correlation.
- Login failures are logged with a masked email; passwords and check-in codes are never logged.
- Exports and generated documents are streamed rather than stored.

---

## 4. Verification

Automated coverage lives in `server/tests/` and runs with:

```bat
npm test
```

The suite uses Vitest + Supertest against a disposable in-memory MongoDB. Each file
starts its own database, seeds real bcrypt hashes, and signs in through the real
login endpoint.

| File | Security-relevant checks |
| --- | --- |
| `api-hardening.test.js` | `Cache-Control: no-store` on successes and 404s, no ETag, no `x-powered-by`, Helmet headers, request-id echo, malformed JSON `400`, form-encoded rejection, CORS allowlist, untrusted-origin `403`, missing-origin behavior |
| `auth.test.js` | `401` without a session, HttpOnly/SameSite cookie, token absent from the body, generic failure messages, email normalization, deactivated account `403`, tampered cookie `401`, `sessionVersion` revocation, logout, teacher/admin `403`, role reload from the database; sign-up creates a pending teacher with no cookie, blocks sign-in until approval, ignores a client-supplied `role`, rejects duplicates and weak input, and notifies active admins |
| `approvals.test.js` | Pending queue, exclusion from the directory and overview totals, approval opening sign-in with an `account_approved` notification, rejection with a note, teacher `403` on both endpoints, invalid decisions, unknown ids, and repeat decisions (`409`) |
| `leave.test.js` | Teacher-only creation, admin-only review, date and length validation, overlap `409`, rejection note requirement, ownership of withdrawals, role-scoped listings |
| `notifications.test.js` | Per-user scoping, cross-account `404`, accurate unread counts, pagination |
| `profile.test.js` | Name/phone-only updates, ignored email/role writes, current-password check, password rotation revoking old cookies, per-user throttling `429` |
| `routineImport.test.js` | PDF and CSV parsing, balanced auto-assignment, conflict warnings, publish replacement with teacher notifications, digest deduplication, teacher `403` on routine and digest endpoints |

Manual checks performed during earlier phases (documented in the README checklists)
cover the browser flows: role-based navigation, the 403 screen, dark theme, session
expiry handling, and export downloads.

---

## 5. Residual risks and follow-ups

| Risk | Why it remains | Suggested action |
| --- | --- | --- |
| No MFA or SSO | Out of scope for this build | Add TOTP or institutional SSO for administrators first |
| Rate limits are in memory | State is per process and resets on restart | Use a shared store (Redis) and keep the API behind a WAF when running more than one instance |
| Leave overlap is check-then-insert | Standalone MongoDB has no cross-document constraint or transactions | Accept for current scale; enable a replica set and use a transaction, or add a unique per-day index collection |
| Requests without an `Origin` are accepted | Non-browser clients do not send it; blocking would break the seed and test tooling | Accepted with `SameSite=Lax` and the CORS allowlist; keep the client and API on the same site |
| Revocation is all-or-nothing | Only `sessionVersion` exists; there is no per-device session list | Add a session collection if per-device sign-out is required |
| The displayed check-in code is not a second factor | The code is shown to the signed-in teacher, so it proves session control and presence in the timetable, not mailbox possession | Accept for this deployment; restore an out-of-band channel (email or authenticator) if policy requires proof of possession |
| No password-complexity history | Only a "must differ from current" check exists | Acceptable for a staff tool; revisit if policy requires history |
| Admin actions are not double-approved | One administrator can change accounts and approve leave | Add a second approver or an approval log if policy requires it |
| Static client headers depend on the host | Helmet only hardens API responses | Configure CSP/HSTS on the static host or CDN serving `client/dist` |
| `TRUST_PROXY` misconfiguration | Wrong setting can collapse client IPs or trust spoofed headers | Set it only when exactly one trusted proxy sits in front of the API |
| Open registration, gated by approval | Anyone who can reach `/register` can create a pending teacher request; it has no access and is invisible until an administrator approves it, so the remaining exposure is queue noise and the reviewer's judgement | Keep the per-IP sign-up limit, review requests in **Teachers → Approvals**, and reject unknown applicants; add institution-domain verification if requests must be filtered before review |
| Routine parsing is heuristic | PDFs from different tools lay text out differently; a misread row could assign the wrong teacher or room | Keep the preview step: every row and auto-fill is shown before publishing, blocked rows cannot be published, and publishing is admin-only |
| Uploads consume process memory | A 5 MB limit and 20-page cap bound each file, but many concurrent admin uploads still share API memory | Acceptable for a staff tool behind authentication; move parsing to a queue or a separate worker if uploads become frequent |
| Approval depends on one administrator | A compromised or careless admin session can approve an account, and a pending request waits until someone signs in | Use a strong admin password, review the queue regularly, and deactivate unexpected accounts; add a second approver for high-security deployments |
| No email verification on sign-up | A person can register with an address they do not own; check-in no longer depends on that address, so the impact is limited to contact accuracy and admin notifications | Add an email-confirmation step before production enrollment if institution policy requires it |

None of these expands a teacher's access beyond their own records or lets an
unauthenticated caller read data.

---

## 6. Production deployment checklist

### Environment

- [ ] `NODE_ENV=production`
- [ ] `MONGODB_URI` points at a MongoDB deployment with authentication enabled and a least-privilege user
- [ ] `JWT_SECRET` is at least 32 random characters (48+ recommended) and stored in a secret manager, not in a file in the web root
- [ ] `CLIENT_ORIGIN` lists exact **HTTPS** origins only, comma-separated
- [ ] `TRUST_PROXY=true` if and only if one trusted reverse proxy terminates TLS
- [ ] `SESSION_TTL_HOURS` matches institutional policy
- [ ] `COLLEGE_TIMEZONE` is correct; attendance windows depend on it
- [ ] Rate-limit variables are set for expected load, including `REGISTER_RATE_LIMIT_WINDOW_MINUTES` and `REGISTER_RATE_LIMIT_MAX` for public sign-up
- [ ] `USE_IN_MEMORY_DB` is `false` (the API refuses to start otherwise)
- [ ] Seed variables (`ADMIN_*`, `SEED_*`, `DEMO_*`, `DEMO_PASSWORD`) are removed after provisioning
- [ ] `RESET_PASSWORD` is passed only for a single reset command and is not stored in a shell profile or service definition

### Network and hosting

- [ ] TLS terminates at the proxy; HTTP redirects to HTTPS
- [ ] The API is not directly reachable from the internet; only the proxy reaches it
- [ ] MongoDB is firewalled to the API host and has authentication enabled
- [ ] HSTS is enabled (Helmet sends it; confirm the proxy does not strip it)
- [ ] The static client is served over HTTPS with a CSP, `X-Content-Type-Options`, and `Referrer-Policy`
- [ ] `client` is built with the production API URL (`VITE_API_URL`) and the dev proxy is not used

### Accounts and data

- [ ] The first administrator is created with `npm run seed:admin`, then the seed variables are removed
- [ ] Public sign-up is either intended for this deployment or the endpoint is blocked at the proxy
- [ ] An administrator monitors **Teachers → Approvals** (and the dashboard pending card) and rejects unknown requests
- [ ] Approved self-registered teachers are reviewed in the admin directory and assigned a department and employee ID
- [ ] A password-recovery procedure using `npm run reset:password` is documented for operators
- [ ] `npm run seed:demo` is **not** run in production
- [ ] Demo passwords from the README are not used anywhere
- [ ] Backups run and a restore has been rehearsed
- [ ] A retention policy exists for attendance audit records and leave decisions

### Operations

- [ ] Logs are aggregated without request bodies, cookies, passwords, or check-in codes
- [ ] `/api/health/ready` is probed by the load balancer; `/api/health` is monitored
- [ ] Alerts exist for repeated `401`/`403`/`429` spikes and for 5xx rates
- [ ] `npm audit --omit=dev` is reviewed on each dependency change; the lockfile is committed
- [ ] A process manager restarts the API and preserves stderr/stdout
- [ ] Rotating `JWT_SECRET` is treated as a global sign-out and is planned
- [ ] Dependency and Node.js versions stay on supported releases

### Before each release

- [ ] `npm test` passes
- [ ] `npm run build` succeeds
- [ ] New endpoints are registered behind `authenticate` and an explicit role check
- [ ] New request input has a Zod schema
- [ ] New responses do not include password hashes, OTP hashes, session tokens, or another teacher's check-in code

---

## 7. Reporting a vulnerability

If you find a security issue, report it privately to the maintainer of your
deployment or your institution's security contact. Include the endpoint, the
steps to reproduce, and the impact. Do not open a public issue with exploit
details, and do not test against production accounts or data you do not own.
