# Learn to Skate — Attendance Check-In

A standalone kiosk app for tracking Learn to Skate attendance. Families scan a
QR code (or type a short code from a printed card) when they arrive, and the
check-in is logged live to Firestore. Staff manage the roster and view
attendance from a separate admin console.

It uses the **same Firebase project** as the CXA Parent Portal
(`cxa-parent-portal`), in two new Firestore collections, so no new Firebase
project setup is required. Staff admin accounts are the same ones already
used for the parent portal (`users/{uid}.role == 'admin'`).

## Pages

- `index.html` — public kiosk screen. Opens the camera and scans QR codes, or
  accepts a typed code as a fallback. No login required.
- `admin.html` — staff-only console: manage the family/skater roster, generate
  & print QR codes, and view a live attendance log with CSV export. Requires
  the same admin login as the parent portal.

## Data model

**`lts_families/{code}`** — one document per family, keyed by a random
6-character code (also encoded into the printed QR).
```
{
  familyName: "The Wolitski Family",
  parentName: "Carlos P.",
  contact: "carlos@coloradoextreme.org",
  students: [ { name: "Vaughn Wolitski", level: "Basic 3" } ],
  active: true,
  createdAt, updatedAt
}
```

**`lts_attendance/{autoId}`** — one document per check-in event (student names
are copied at check-in time so history stays intact even if the roster later
changes).
```
{
  familyId: "7F3K9Q",
  familyName: "The Wolitski Family",
  students: [ { name: "Vaughn Wolitski", level: "Basic 3" } ],
  date: "2026-07-01",       // America/Denver, YYYY-MM-DD
  checkedInAt: <server timestamp>
}
```

A family can only be checked in once per calendar day — scanning again just
shows "Already checked in" instead of creating a duplicate record. Staff can
still log an extra manual check-in from the admin console if needed.

## Required setup: Firestore rules

The kiosk (`index.html`) is unauthenticated by design (self-check-in, no
login), so it needs limited public access to look up codes and log
check-ins. Add the following to your existing Firestore rules in the
[Firebase console](https://console.firebase.google.com/project/cxa-parent-portal/firestore/rules)
— **alongside**, not replacing, your current rules for the parent portal:

```
function isAdmin() {
  return request.auth != null &&
    exists(/databases/$(database)/documents/users/$(request.auth.uid)) &&
    get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'admin';
}

match /lts_families/{familyId} {
  // Public read so the kiosk can look up a scanned code without logging in.
  allow read: if true;
  allow write: if isAdmin();
}

match /lts_attendance/{recordId} {
  allow read: if isAdmin();
  allow create: if
    request.resource.data.familyId is string &&
    exists(/databases/$(database)/documents/lts_families/$(request.resource.data.familyId)) &&
    request.resource.data.keys().hasOnly(['familyId','familyName','students','date','checkedInAt']);
  allow update, delete: if isAdmin();
}
```

**Note on scope:** allowing public `read` on `lts_families` and public
`create` on `lts_attendance` is what makes an unattended, un-authenticated
kiosk possible. It means anyone with the kiosk URL could technically read
family/skater names or spam check-in writes (the `create` rule above at least
requires a real, existing family code). If that's a concern for your
deployment, consider adding Firebase App Check, or moving the write behind a
Cloud Function later — that's a bigger change outside this app's current
scope.

## Using it

1. **Staff:** open `admin.html`, sign in with an admin account, go to
   **Roster → Add Family**, enter the family/skater info, and save. A unique
   code is generated automatically.
2. Click **QR Code** on that family's row to view/print/download a card with
   their QR code and the short code as text (for manual entry).
3. Hand out the printed card, or let the family save/screenshot the QR.
4. **Families:** on future visits, open the kiosk (`index.html`) on the
   front-desk tablet/laptop and scan the code, or type the short code if
   scanning fails.
5. **Staff:** the **Attendance** tab shows who has checked in for any given
   day in real time, with a CSV export for record-keeping.

## Deployment

This folder is part of the same GitHub Pages site as the parent portal, so
once merged it will be reachable at `/Carlos/learn-to-skate/` and
`/Carlos/learn-to-skate/admin.html`. Camera access requires HTTPS, which
GitHub Pages provides.
