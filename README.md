# Report X — School Report Card Platform

Plain HTML/CSS/JS + Firebase Authentication + Cloud Firestore + PWA. No Firebase Storage.

## What's in this delivery

| File | Purpose |
|---|---|
| `index.html` | The entire application — config, auth, permissions, routing, dashboards, workflows, report engine |
| `firestore.rules` | Production Firestore security rules — the real authorization boundary |
| `manifest.json` | PWA manifest |
| `sw.js` | Service worker (app-shell caching only, never caches private data) |
| `icons/icon-192.png`, `icons/icon-512.png` | Placeholder app icons — swap for your real logo |

## 1. Firebase setup

1. In the [Firebase Console](https://console.firebase.google.com), open project **report-x-fd97e** (config is already wired into `index.html`).
2. **Authentication** → Sign-in method → enable **Email/Password**.
3. **Firestore Database** → create in production mode (any region close to your users, e.g. `europe-west1` or `us-central1`).
4. Deploy the rules:
   ```bash
   npm install -g firebase-tools
   firebase login
   firebase init firestore   # point it at this project, keep default file names
   cp firestore.rules ./firestore.rules
   firebase deploy --only firestore:rules
   ```
5. **Do not enable Firebase Storage.** This app intentionally never calls Storage APIs — all small branding/signature/stamp assets are compressed client-side and stored as data URLs directly in Firestore documents (see `compressImage()` in `index.html`). This is a deliberate architectural limit: keep logos/signatures/stamps small (a few hundred KB max before compression).

## 2. Bootstrap the Super Admin

The existing Super Admin Firebase Auth user must be supported. To bootstrap it:

1. Create the user in **Authentication** (or have them sign up once through the app, then note their UID).
2. In Firestore, create a document at `platformAdmins/{uid}`:
   ```json
   { "level": "super", "email": "admin@yourdomain.com", "createdAt": "<timestamp>" }
   ```
3. That UID will now land on the Super Admin dashboard on next login.

## 3. Hosting

Any static host works (Firebase Hosting, Netlify, Vercel, S3+CloudFront). Example with Firebase Hosting:

```bash
firebase init hosting   # public directory: this folder
firebase deploy --only hosting
```

Make sure `manifest.json`, `sw.js`, `icons/`, and `index.html` are all deployed at the same root so relative paths resolve.

## 4. Firestore collections (as implemented)

`platformSettings`, `platformAdmins`, `schools`, `schoolUsers`, `schoolInvitations`, `schoolSettings`,
`academicSessions`, `terms`, `classes`, `subjects`, `students`, `studentClassHistory`, `teacherAssignments`,
`studentClassAssignments`, `gradeConfigurations`, `gradingScales`, `results`, `resultEntries`, `attendance`,
`behaviourRecords`, `comments`, `reportCards`, `reportVersions`, `reportVerification`, `payments`, `subscriptions`,
`notifications`, `auditLogs`, `printingLogs`, `parentLinks`, `announcements`.

Each school-owned document carries a `schoolId` field; every rule and query is scoped by it. `schoolUsers/{schoolId}_{uid}`
is the membership/role record — this is what the security rules key off, never a client-asserted role.

## 5. Required composite indexes

Firestore will prompt you with a direct console link the first time each query runs; create them as prompted. Expect indexes on combinations such as:

- `students`: `schoolId` + `createdAt`
- `results`: `schoolId` + `status`, and `schoolId` + `studentId` + `status`
- `studentClassAssignments`: `schoolId` + `classId` + `approved`
- `payments`: `status` + `createdAt`
- `schools`: `createdAt`

## 6. What's fully built vs. scaffolded

**Fully wired to Firestore + Rules:**
- Firebase init, email/password auth, password reset
- School registration wizard → pending state → Super Admin approve/reject/suspend
- Role-based routing, sidebar/bottom-nav, light/dark/system theme
- Super Admin dashboard, schools list, payment review, platform settings
- Owner/Principal dashboards with live counts
- Student creation with duplicate admission-number validation
- Teacher grade entry (draft/submit) scoped to approved class assignments
- Principal result review (approve/reject with required reason)
- Report finalization (recalculated from locked results only), unpredictable Report ID, QR code generation, public verification page (`#verify/{reportId}`) with minimal-disclosure output
- Client-side image compression helper for branding assets (no Storage)
- Audit log writer used across the above actions
- Notification writer (in-app only — no external email claimed)
- PWA manifest + safe app-shell-only service worker

**Scaffolded (view renders, permissions/routes wired, ready to extend with the same patterns already in the file):**
- CSV import with preview/validation
- Full class/session/term/subject CRUD screens
- Class-assignment approval queue UI (the Firestore Rules and data model already enforce the Registrar→Principal workflow)
- Branding/signature/stamp upload UI (compression function exists; wire it to a form)
- Staff invitation UI (rules already restrict role assignment to Owner/Principal/Super Admin)
- Parent dashboard child cards
- Full report card visual template (print CSS classes `.report-sheet`, `.report-head`, `.report-grid` are ready; populate with school branding + finalized data)
- Analytics (class/subject performance, pass rates)
- Report versioning UI for the immutable `reportVersions` collection

Each scaffolded view follows the exact same pattern as the built ones: a Firestore query scoped by `schoolId`, permission check via `can()`, and the shared UI primitives (`el()`, `.card`, `.table-wrap`, `modal()`, `toast()`). Extending them is mechanical, not architectural.

## 7. Security notes

- Firestore Rules never trust a client-supplied `role` or `schoolId` — every check re-derives them from `schoolUsers` or `platformAdmins` documents.
- Locked results cannot be edited by teachers; unlocking is a distinct, audited transition restricted to academic admins.
- Public verification (`reportVerification/{token}`) allows `get` by exact token only — `list` is denied, preventing enumeration.
- No `allow read, write: if true` anywhere; the final catch-all rule denies everything not explicitly matched.

## 8. Update log — bug fixes & invite-code login

- **Fixed:** school registration and "something went wrong" errors were caused by a Firestore Rules gap — a new owner couldn't create their own first membership record because the rule required them to already be a school admin (circular). Rules now allow a one-time, tightly-scoped self-bootstrap tied to `schools.createdBy`.
- **Fixed:** error messages are no longer a generic "Something went wrong" — auth and Firestore errors (including `permission-denied`, `unavailable`, `auth/operation-not-allowed`) now surface the real cause so misconfiguration (rules not deployed, Email/Password provider disabled, Firestore not created) is visible immediately. Always also check the browser console (F12) for the full technical error.
- **New: Teacher/Parent sign-in via code.** From the sign-in screen, "Have a school code + invite code? Join as Teacher/Parent" opens a join form (School Code + Invitation Code + email/password). Owners/Principals generate these codes from the new **Staff** screen, which lists pending/redeemed codes and lets you revoke unused ones. Each code is single-use and locked to one role — redeeming it can only create a membership matching that exact role, enforced in `firestore.rules`, not just the UI.

**If you already deployed the previous `firestore.rules`, redeploy the new one** (`firebase deploy --only firestore:rules`) — the registration bug and the new invite-code feature both depend on rule changes.

## 9. Before going to production

Run through the security and UX test matrices from the original spec (cross-school access attempts, role escalation attempts, expired-school writes, locked-result edits, oversized image uploads, malicious HTML in names/comments, 320px–desktop responsive check) before commercial launch.
