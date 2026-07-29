# CareDesk — Appointment Booking Register

A multi-clinic appointment booking register: **Doctor Appointments, Test
Bookings, Home Collection, and Dental Appointments** — with role-based staff
accounts, Excel import/export with downloadable templates, a recycle bin for
deleted bookings *and* deleted staff, register-wise bulk delete for super
admins, a full audit trail, precomputed reports/dashboard for fast loading
even with a lot of data, and a live dashboard.

Single HTML file (`index.html`), no build step, no framework — just open it in
a browser once Firebase is connected. Firebase JS SDK, SheetJS (Excel
import/export) and Chart.js are all loaded from CDNs, and only fetched when
actually needed (SheetJS/Chart.js are never on the critical path for the
login screen).

---

## 1. Create your Firebase project

1. Go to the [Firebase Console](https://console.firebase.google.com) → **Add project**.
2. Inside the project, go to **Build → Authentication → Get started**, and enable the
   **Email/Password** sign-in provider. (Staff sign in with email + password —
   there's no public sign-up screen by design; accounts are created by an
   admin/super admin from inside the app.)
3. Go to **Build → Firestore Database → Create database**. Start in
   **production mode** (the included `firestore.rules` will lock it down properly).
4. Go to **Project settings → General → Your apps → Add app → Web (`</>`)**.
   Register the app (no need for Firebase Hosting unless you want it) and
   copy the `firebaseConfig` object it gives you.

## 2. Connect the app

Open `index.html`, search for `firebaseConfig` near the top of the
`<script type="module">` block, and replace the placeholder values with the
real config from step 1.4:

```js
const firebaseConfig = {
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "..."
};
```

Save the file. Until this is filled in, opening `index.html` shows a
"Firebase config needed" screen instead of the login page.

**Stuck on the loading spinner and it never reaches the login page?** This
almost always means the page can't reach Firebase's servers from wherever
it's being viewed — most commonly because it's being opened inside a
sandboxed in-app preview instead of a real browser. Download `index.html`
and either open it directly (double-click) or host it (Firebase Hosting,
Netlify, a plain web server), then reload. If it still hangs after ~8
seconds the page itself will say so; open the browser console (F12) for the
exact error either way.

## 3. Deploy the security rules

`firestore.rules` (included) enforces the role/clinic/register permission
model at the database level — not just in the UI. In the Firebase Console,
go to **Firestore Database → Rules**, paste in the contents of
`firestore.rules`, and click **Publish**.

(If you use the Firebase CLI instead: `firebase deploy --only firestore:rules`.)

## 4. Create your first Super Admin

There's no public sign-up — the very first account has to be created by hand
directly in Firebase, once:

1. **Authentication → Users → Add user.** Enter an email and a password.
   Click the new user and copy their **User UID**.
2. **Firestore Database → Start collection.** Collection ID: `users`.
   Document ID: paste the **User UID** from step 1. Add these fields:

   | Field     | Type    | Value               |
   |-----------|---------|---------------------|
   | `name`    | string  | e.g. `Admin`        |
   | `email`   | string  | same email as step 1|
   | `role`    | string  | `superadmin`        |
   | `deleted` | boolean | `false`             |

3. Save. Open `index.html` and sign in with that email/password — you're in
   as Super Admin. From there, create your first Clinic, then a Clinic Admin
   for it (who can in turn create Agents), all from inside the app.

## 5. Host it

Any static host works — Firebase Hosting, Netlify, a plain web server, or
just opening the file locally for testing. It's a single self-contained HTML
file; there's nothing to build.

## 6. (Optional) Complete hard delete — deploy the Cloud Function

Deleting a clinic or purging a staff member from the recycle bin already
deletes everything in Firestore straight from the app. The one thing a
client-side app is *never* allowed to do — by Firebase's own design, not a
bug — is delete *another person's* sign-in (Auth) account; only trusted
server code can do that. The included `functions/` folder is that trusted
server code. It's optional: without it, hard-deleted staff simply lose all
access and data immediately (their empty sign-in shell just sits unused) —
deploy this only if you want that shell gone too.

**Requires the Blaze (pay-as-you-go) plan** — Cloud Functions can't deploy on
the free Spark plan. In practice this stays free: the free quota (2M
invocations/month) is far more than clinic-deletion volume would ever use.

1. Install the [Firebase CLI](https://firebase.google.com/docs/cli) if you don't have it: `npm install -g firebase-tools`
2. In the Firebase Console, upgrade the project to **Blaze** (Project settings → Usage and billing).
3. From a terminal, in the same folder as `index.html`, `firebase.json`, `.firebaserc` and `functions/`:
   ```
   firebase login
   firebase deploy --only functions
   ```
   (`.firebaserc` already points at this project — `caredesk-65c0c` — so there's nothing else to configure unless you're using a different project.)
4. That's it — no changes needed in `index.html`. It already calls these
   functions when deleting a clinic or purging a user, and simply skips that
   step (Firestore data still gets deleted normally) if they're not deployed.

---

## How the pieces fit together

- **Clinics** — each clinic has its own doctors, tests, bookings and staff.
  A Super Admin can manage every clinic from **Clinics**; entering a clinic
  drops them into that clinic's dashboard just like a Clinic Admin sees it.
- **Roles** — Super Admin (all clinics) → Clinic Admin (their one clinic,
  full access) → Agent (their one clinic, only the registers they've been
  granted — set per-agent under **User Management**).
- **Quick status change** — the Status column on every register's list is a
  live dropdown, not just a label — change Booked → Completed/Cancelled/
  Rescheduled right from the list, no need to open the full edit form. Every
  change is still logged to that booking's History either way.
- **Recycle Bin** — deleting a booking or a staff account never deletes it
  outright; it's soft-deleted and shows up in that register's (or Users')
  Recycle Bin. **Only Admins and Super Admins can open a Recycle Bin** —
  Agents can send something *to* it (their usual delete button) but can't
  restore or purge from it, matching the original app's pattern. Deleted
  staff accounts lose sign-in access immediately (enforced in
  `firestore.rules`, not just the UI).
- **Hard delete** — two things are genuinely, permanently removed from
  Firestore, both restricted to Admin/Super Admin:
  - **Purge** — inside a Recycle Bin, permanently deletes that one record.
  - **Delete Clinic Entirely** — Super Admin → Clinics → **Manage** →
    Danger Zone. Wipes the clinic's bookings, doctors, tests and staff, then
    the clinic itself. Super Admin only.
  Agents can create/edit/soft-delete within their registers, but the
  `delete` operation itself is blocked for them at the database level
  (`firestore.rules`), not just hidden in the UI.
- **Bulk Delete** — Super Admin → Clinics → **Manage** on a clinic → pick a
  register and a date range, or clear an entire Doctors/Tests master list in
  one go. Also a *permanent* delete, separate from the per-record soft-delete
  everyone else uses.
- **History** — every booking has a full, append-only audit trail: who
  created it, every field anyone changed (old value → new value), and every
  soft-delete/restore, each with who and when. Open any existing booking and
  click **History** to see it. Nobody — not even an Admin — can edit or
  delete a history entry once it's written (`firestore.rules` blocks it
  outright), so it stays trustworthy.
- **Settings** — clinic hours and per-register time-slot length, the list of
  booking sources, and dental treatment types are all editable per clinic
  under **Settings**.
- **Creating staff** — creating a Clinic Admin or an Agent, you (the creator)
  set their password directly on the spot — no temporary password to relay
  separately. Share the email + password with them yourself; the "My
  Account" menu (click your name in the sidebar) lets anyone change their
  own password afterward.
- **Import** — every register's list view and the Masters (Doctors/Tests)
  screen have an Import button (Admin/Super Admin only). It always starts
  with **Download Template** — an .xlsx with the exact expected columns and
  one example row — so there's no guessing the format. Re-upload the filled
  template and it's validated row by row before anything is written; rows
  with problems are listed with the reason and skipped, everything else
  imports. Master imports match by Name (existing entries are updated, new
  ones added); booking imports always come in with status "Booked".
- **Performance** — Dashboard and the Reports Summary tab read from small
  precomputed total documents (`meta/dashboardSummary` and
  `monthlySummaries/{register}_{month}`) instead of scanning every booking,
  so they stay fast regardless of how much history a clinic has. These
  totals update themselves automatically — every booking write updates them
  in the same batch, at no extra cost — but if numbers ever look off (e.g.
  after data was edited directly in the Firebase console), **Settings →
  Recalculate Summary** rebuilds them from scratch.

## Known limitations, by design

- **Purging a staff account from the recycle bin, or deleting a whole
  clinic,** deletes the Firestore data straight from the app either way —
  and also deletes the underlying Firebase *Auth* sign-in accounts too, if
  the optional Cloud Function (step 6 in setup) is deployed. Without it,
  Firestore data is still fully removed and access is revoked immediately
  either way (no profile = can't sign in); the Auth account shell just isn't
  physically deleted until that function runs.
- **Deleting a clinic, or purging a single booking,** removes that document,
  but its `history` subcollection isn't recursively deleted by the client
  SDK (Firestore has no client-side recursive delete). Those orphaned
  history entries are unreachable through the app and harmless to leave; a
  Cloud Function (or the `firebase firestore:delete --recursive` CLI command)
  can clean them up if you'd rather they not exist at all.
- **PDF export** isn't wired up yet — the Excel export (Reports and each
  register's list) is. The data layer is already shaped so a PDF export
  button could reuse the same row data later.
- Report and list queries are scoped by date range (and capped) rather than
  scanning full history, which keeps reads cheap — narrow the date range for
  very high-volume periods.
