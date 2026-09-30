# Antigravity Prompt: Build the Club Activity Management System (CAMS) backend and wire the existing frontend

Paste everything below into Antigravity. Keep `index.html` and the `assets/` folder in the project root.

---

## 1. What you are working on

I have a finished **frontend-only** prototype in `index.html` (single file, HTML + CSS + vanilla JS) for the **Club Activity Management System**, an institutional portal for NIT Trichy's Office of Student Affairs. Right now everything is hardcoded sample data and sign-in accepts anything.

Your job is to **build the real system behind it and connect it to the existing screens**.

### Golden rules (do not break these)
1. **Do not redesign the UI.** Keep the exact same look: layout, colors, fonts, spacing, class names, animations. If you must add a new screen or component, build it from the existing CSS classes and design tokens so it looks like it was always part of the app.
2. **Work inside my existing code structure.** Do not rewrite `index.html` from scratch. Edit in place. Keep the `screen` / `view` switching, the `data-role-club` / `data-role-admin` visibility system, and the modal helpers. If you split the inline `<script>` and `<style>` into `/public/js/*.js` and `/public/css/*.css`, move the code as it is, without restyling.
3. **Explain things in plain language.** When you finish a phase, tell me in simple words what you built, how to run it, and how to test it.
4. Work **phase by phase** (Section 8). Stop after each phase and let me test before moving on.

### Design tokens (already in the file; reuse, never replace)
- Navy: `--navy-950 #081426`, `--navy-900 #0d2440`, `--navy-800 #15325a`, `--navy-700 #1c3f6e`
- Gold: `--gold-600 #b8863b`, `--gold-500 #caa056`, `--gold-100 #f5ead2`
- Background `--bg #eef1f5`, surface `#ffffff`, border `#d7dce3`, text `#1c2530`, muted `#5b6675`
- Status colors: success `#1c7a4c`, danger `#b3261e`, warning `#a6650d`, info `#28587a` (each with a `-bg` tint)
- Radius `4px`. Headings: Georgia (serif). Body: Arial. No other fonts, no gradients or rounded "app-style" cards beyond what already exists.
- Existing components to reuse: `.btn` (`btn-primary`, `btn-outline`, `btn-gold`, `btn-danger-outline`, `btn-success-outline`, `btn-sm`, `btn-block`), `.badge` (`success/warning/danger/info/neutral`), `.card`, `.stat-card`, `table.data`, `.kanban`, `.event-card`, `.report-card`, `.modal`, `.role-chip`, `.activity-list`, `.ticker` notice board, `.field`.

### Assets
The HTML references `assets/nitt-logo.png` and `assets/admin-block.JPG`. Create the `assets/` folder and tell me to drop those two files in. If a file is missing, the page must still look fine (no broken-image icon).

---

## 2. What the landing page promises (this is the requirements list)

> "One system for every club, event and approval on campus. Register a new club, track member activity, run events end-to-end, and generate reports — all from a single institutional portal used by Super Admins, Club Presidents, Faculty Advisors and Members alike."

So the finished product must genuinely deliver:
- **Register a new club** (public request form, then Super Admin verification, then credentials by email)
- **Track member activity** (roles, attendance, tasks, participation)
- **Run events end-to-end** (create, approve, publish, register, attend, feedback, certificates)
- **Generate reports** (PDF and Excel)
- **Four roles:** Super Admin, Club President, Faculty Advisor, Member / Volunteer

---

## 3. Tech stack

- **Backend:** Node.js + Express
- **Database:** SQLite (`better-sqlite3`), file at `/data/cams.db`, with a `schema.sql` and a `seed.js`
- **Auth:** email or user ID + password, `bcrypt` hashing, session cookie (`express-session`) or JWT in an httpOnly cookie. Add login rate limiting.
- **Other libraries:** `multer` (uploads), `nodemailer` (emails; use a console/Ethereal fallback in dev), `qrcode` (attendance QR), `pdfkit` (PDF reports and certificates), `exceljs` (Excel reports), `helmet`, `express-validator`
- **Frontend:** stays vanilla HTML/CSS/JS served statically by Express. No React or frameworks.
- Provide `.env.example`, `npm start`, `npm run seed`, and a short README.

---

## 4. Roles and permissions

The login card has 4 tabs. Right now 3 of them share the same "club" view, so **make them genuinely different**:

| Feature | Super Admin | Club President | Faculty Advisor | Member / Volunteer |
|---|---|---|---|---|
| Review, approve, reject or request info on club requests | Yes | No | No | No |
| Clubs directory (all clubs) | Yes | No | No | No |
| Audit log | Yes | No | No | No |
| Manage members and roles in own club | No | Yes | View only | No |
| Create events (draft, then submit for approval) | No | Yes | No | No |
| Approve club events and budgets | No | No | Yes | No |
| Create tasks and assign volunteers | No | Yes | View only | Own tasks only |
| Publish announcements | No | Yes | Yes | No |
| Register for events, mark attendance via QR | No | Yes | No | Yes (register, scan) |
| Reports | All clubs | Own club | Own club | No (own certificates only) |
| Certificates and feedback | Issue and view | Issue | View | Receive and give feedback |

Rules:
- Enforce permissions **on the server** for every route (middleware like `requireRole(...)` and `requireClubMember(...)`). Hiding buttons in the UI is not security.
- Roles inside a club: Faculty Advisor, President, Vice President, Coordinator, Core Member, Volunteer, General Member (these already appear as `.role-chip`s). Least privilege: each sees only what it needs.
- Keep the existing `body.role-superadmin` / `body.role-club` mechanism, and add `body.role-advisor` and `body.role-member` classes with matching `data-role-*` attributes so each role sees the right sidebar and views.

---

## 5. Modules to build (each maps to an existing screen or view)

### 5.1 Authentication (`#landing`)
- Real sign-in using the selected role tab. Reject the login if the account's role does not match the tab.
- "Forgot password?" flow: email a reset link (time-limited token).
- "Keep me signed in" checkbox extends the session.
- Sign Out clears the session. Every protected API returns 401 when logged out. Show a friendly inline error in the login card (use the `--danger` color).
- Remove the "frontend interface preview" demo note only once real auth works.

### 5.2 Club creation request (`#request` and `#confirm`)
- Form fields exactly as they are: club name, description, objective, category (Technical, Cultural, Literary, Sports, Professional / Career, Social Service), proposed faculty advisor, supporting documents (proof of interest, draft constitution, past activities), other details.
- Validate on client and server. File upload with type and size limits (PDF, DOCX, JPG, PNG, max 5 MB each).
- On submit, generate a real request ID in the format `REQ-2026-00147` (year plus a running number), save it, and show it on the confirmation screen with the "Pending Review" badge.
- Email confirmation to the requester.

### 5.3 Verification queue (Super Admin: `#view-verification`, `#modal-review`)
- Table lists real requests with the document count (e.g. `4 / 4`) and a status badge: Pending, Under Review, Needs Info, Approved, Rejected.
- The review modal uses the existing checklist: check documents, validate club does not already exist, verify proposed faculty advisor, check name uniqueness, additional clarification. Save the checkbox state per request, plus remarks.
- **Approve** creates the club, creates the president and advisor accounts if needed, emails credentials (temporary password, forced change at first login). **Reject** and **Ask for More Info** email the remarks to the requester.
- Every decision writes to the audit log.

### 5.4 Clubs directory (Super Admin)
- Real list: name, category, president, advisor, member count, status. Add search. Allow activating and deactivating a club.

### 5.5 Members and roles (`#view-members`, `#modal-member`)
- List, search, add (sends an invitation email), edit role, deactivate. The Faculty Advisor and President are protected roles (cannot be removed by a lower role).

### 5.6 Events (`#view-events`, `#modal-event`)
- Lifecycle: **Draft, Submitted, Approved, Published, Completed** (and Rejected). The existing cards already show Draft / Approved / Completed, so extend with the rest using the same badges.
- "Save as Draft" and "Submit for Approval" in the modal. The Faculty Advisor approves. Only approved events can be published.
- Replace the free-text date field with proper validation but keep the same input styling (a `datetime-local` input styled with the existing `input` CSS is fine).
- Registration count, capacity, and student registration for published events.

### 5.7 Attendance
- Generate a unique QR code per event. Members scan or enter a code to mark attendance. Also allow manual marking by the President. Prevent duplicate or late marking.

### 5.8 Tasks and volunteers (`#view-tasks`, kanban)
- Real kanban columns (To Do, In Progress, Completed). "+ New Task" opens a modal (same style as the others) with title, assignee, due date, and the related event. Move tasks between columns (buttons or drag and drop).
- Volunteers can request to help with an event; the President accepts or declines (this appears in the Recent Activity feed).

### 5.9 Announcements (`#view-announcements`)
- Publish to club members as both an **email** and an **in-app notification**. Show "Recently Sent" from the database.

### 5.10 Notifications and notice board
- The bell count in the top bar (`#notifCount`) reads real unread notifications. Add a small dropdown using existing card styling.
- The scrolling **Notice Board** ticker is populated from published announcements and approvals.

### 5.11 Certificates and feedback
- After an event is marked Completed, auto-generate PDF certificates for attendees, with a unique verification code and a public verify page. Members can download their own.
- Attendees can rate the event and leave suggestions. The Feedback Report aggregates them.

### 5.12 Reports (`#view-reports`)
- Six cards: Club-wise, Event, Attendance, Member, Certificate, Feedback. The **PDF** and **Excel** buttons must download real files, filtered by role (Super Admin sees all clubs, President and Advisor see their own club).

### 5.13 Audit log (Super Admin)
- Record who did what, when, and in which module (logins, approvals, role changes, event actions, report downloads). Read-only, newest first, with pagination.

### 5.14 Dashboards (`#view-overview`)
- Stat cards (Active Members, Upcoming Events, Pending Tasks, Certificates Issued for clubs; Pending Requests, Active Clubs, Flagged Documents, Members Across Clubs for admin) must be computed from the database, not hardcoded. The welcome text uses the logged-in user's real name and club.
- Add simple Overview pages for Faculty Advisor (pending event approvals) and Member (my events, my tasks, my certificates) using the same `stat-card` and `card` classes.

---

## 6. Database (starting point; refine as needed)

`users`, `clubs`, `club_members` (user_id, club_id, role, status), `club_requests`, `request_documents`, `request_checklist`, `events`, `event_registrations`, `attendance`, `tasks`, `volunteer_requests`, `announcements`, `notifications`, `certificates`, `feedback`, `audit_log`, `password_resets`.

Use foreign keys, indexes on lookups, and parameterized queries only (no string-built SQL).

**Seed data:** reuse the sample data already in `index.html` so the demo looks identical on first run: Code Culture (Arjun Mehta, president; Dr. R. Sharma, advisor; Sneha Kulkarni, Rohan Gupta, Priya Nair, Karthik S, Fatima Sheikh), Robotics & Automation Club, Finance & Investment Cell, Drama Circle, the events, tasks, announcements, requests, and the Super Admin "Rekha Iyer". Create one demo login per role and list the credentials in the README.

---

## 7. Frontend wiring instructions

- Add a small `api.js` helper (`fetch` wrapper with error handling and a loading state).
- Replace hardcoded table rows and cards with rendering from API data, **using the same HTML structure and classes** so it looks identical.
- Add empty states (a plain muted line inside the existing card) and loading states.
- Keep `goTo()`, `showView()`, `showModal()` and `hideModal()` working. Add real submit handlers to the existing buttons instead of replacing them.
- After login, restore the session on page refresh (call `/api/me`, then jump to the app screen).
- Keep it responsive as it is now (sidebar collapses under 820px, grids collapse under 900px and 800px).
- Accessibility: keep the existing focus outlines, add `aria-live` on error messages, and keep the `prefers-reduced-motion` handling.

---

## 8. Build order (stop and report after each phase)

1. **Foundation:** project setup, Express serving `index.html`, SQLite schema and seed, `.env`, README.
2. **Auth and roles:** login, logout, session restore, role middleware, forgot password. Role-specific sidebars for all 4 roles.
3. **Club request and verification:** public form with uploads, request IDs, Super Admin queue and review modal, approval creating the club and accounts, emails, audit log.
4. **Club workspace:** members and roles, events (full lifecycle plus Advisor approval), tasks kanban, announcements, notifications.
5. **Attendance, certificates and feedback:** QR attendance, PDF certificates with verification page, feedback.
6. **Reports and dashboards:** live stat cards, six reports in PDF and Excel.
7. **Hardening:** validation, rate limiting, `helmet`, file checks, error pages, a basic automated test for auth and permissions, final README.

---

## 9. Definition of done
- I can run `npm install`, `npm run seed`, `npm start` and use every role end to end.
- The UI is **visually identical** to the current `index.html` (same colors, fonts, layout).
- No permission can be bypassed by calling the API directly.
- No hardcoded sample data remains in the HTML except empty-state text.
- Every action that matters shows up in the audit log.

Before you start, read `index.html` fully, summarize your understanding and the plan in plain language, and list anything unclear. Then begin Phase 1.
