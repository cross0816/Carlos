# CXA Athlete Hub

A training hub for Colorado Extreme **Sports Academy** athletes, parents,
and coaches. Coaches share training analysis (with video), track athlete
progress with skill assessments, and message athletes and parents directly.
Athletes get a home feed, a progress tracker, and a video/drill library to
grow their knowledge of the game.

Like the Learn to Skate app, it's plain static HTML hosted on GitHub Pages
and uses the **same Firebase project** (`cxa-parent-portal`) — Firebase Auth
for logins and new `ah_*` Firestore collections for data. No new Firebase
project or server is required.

The academy is expanding beyond hockey, so the data model has room for
multiple sports (an athlete's `sport` + `otherSports` fields) even though
**Hockey is the only sport wired up today** — the header's sport switcher
is a placeholder until more sports launch.

## Pages

- `index.html` — the **Athlete Hub** for athletes and parents. Sign in or
  create an account with a 6-character **athlete code** from a coach. Tabs:
  - **Home** — feed of announcements plus training analysis shared with the
    athlete (or with everyone), including embedded video.
  - **My Progress** — a **Player Portfolio** hero (sessions, current streak,
    skills growing since last check-in, and badges earned — see below),
    this week's levels for all 12 skills as a compact card grid, a
    week-by-week development heatmap once 2+ assessments exist, and full
    assessment history with coach notes. Parents with more than one athlete
    get a picker.
  - **Library** — coach-curated videos and drills, filterable by type,
    category, and search.
  - **Messages** — direct messaging with coaches (live updates).
- `coach.html` — the **Coach Console** (coach/admin accounts only):
  - **Dashboard** — a daily-briefing home screen: a greeting, the day's date
    and program/season name (click to rename), three roster stats (total
    athletes, updated this week, need an update), and a card per athlete
    showing their age, weakest skill ("Focus"), other sports, and last
    session date. Each card opens the athlete's full profile, or jumps
    straight to a new assessment via its **Update** button.
  - **Training Feed** — publish analysis posts (title, notes, YouTube link)
    to all athletes or specific ones; edit or delete old posts.
  - **Library** — add/edit/delete videos and drills (category + level).
  - **Messages** — chat with any registered athlete or parent.
  - **Announcements** — broadcast to everyone in the hub; editable/deletable.

### Skill assessments

Each assessment rates 12 skills — Skating, Stickhandling, Passing, Shooting,
Puck Control, Hockey IQ, Mental Game, Sportsmanship, Athletic Movement,
Confidence, Focus, and Effort & Attitude — on a five-stage scale:
**Beginning → Developing → Improving → Consistent → Advanced**. Starting a
new assessment pre-fills every skill with the athlete's most recent levels,
so a coach only has to touch what changed.

### Streak & badges

Computed automatically from assessment history — there's no separate
coach workflow to fill these in:
- **Sessions** — total assessments logged.
- **Streak** — consecutive calendar weeks (Sunday–Saturday) with at least
  one assessment, counted back from the most recent one.
- **Skills Growing** — skills whose stage moved up between the two most
  recent assessments.
- **Badges** — First Session, 5 Sessions, 10 Sessions, 3-Week Streak, Most
  Improved (any skill up 2+ stages from the athlete's first assessment to
  their latest), and Reached Advanced. The hero shows the count earned;
  extending this to a per-badge display (or letting coaches award custom
  ones) is a natural next step if it'd be useful.

The Coach Console's Dashboard tab is still the older list-view (Phase 2 of
the redesign brings its own Weekly Metrics/Season tabs and the "View as
Parent" preview) — the Player Portfolio above is the athlete/parent side
only for now.

Videos are YouTube embeds — upload clips as **unlisted** YouTube videos and
paste the link. Nothing is stored in Firebase Storage, so there are no
storage costs or upload limits to manage.

## How accounts work

1. A coach adds an athlete in the Coach Console → the app generates a
   6-character **athlete code** (same idea as the Learn to Skate family
   code).
2. The athlete (and each parent) goes to the Athlete Hub → **Create
   Account**, picks *Athlete* or *Parent*, and enters the code.
   - An athlete account **claims** the code (one athlete login per code;
     a coach can unlink it from the roster if they need to redo it).
   - Any number of parent accounts can link to the same code. Coaches can
     see and remove parent links from the athlete detail view.
3. **Coach accounts are created by an admin**, the same way parent-portal
   admins are set up today: create the user in Firebase Auth (or have them
   register once), then in Firestore set `users/{uid}.role` to `'coach'`
   (existing `'admin'` users also have full access). On first login to the
   Coach Console the coach is added to `ah_staff` so athletes can find them
   in the "Message a Coach" picker.

## Data model

All collections are prefixed `ah_` so they sit safely alongside the parent
portal and Learn to Skate collections.

**`users/{uid}`** — shared with the parent portal. This app adds
self-registered docs shaped like:
```
{ name, email, role: 'athlete' | 'parent', athleteId: '7F3K9Q' }   // athlete
{ name, email, role: 'parent', athleteIds: ['7F3K9Q'] }            // parent
```
Coaches/admins keep their existing docs with `role: 'coach' | 'admin'`.
To link a parent to a *second* athlete, an admin/coach edits their
`athleteIds` array (and the athlete doc's `parentUids`) — self-service
linking is limited to one athlete at signup to keep the security rules
tight.

**`ah_athletes/{code}`** — one doc per athlete, keyed by their code:
```
{ name, birthYear, position, team, active: true,
  sport: 'hockey', otherSports: ['Golf', 'Lacrosse'],
  uid: <athlete's auth uid or null>, parentUids: [<uid>, ...],
  createdAt, updatedAt }
```
`sport` is always `'hockey'` today (see the multi-sport note above);
`otherSports` is a free-text list the coach types in, shown on the
dashboard as "also Golf, Lacrosse".

**`ah_posts/{id}`** — training analysis:
```
{ title, body, videoUrl, audience: 'all' | 'select',
  athleteIds: [], athleteNames: [], authorUid, authorName, createdAt }
```

**`ah_assessments/{id}`** — one skill assessment:
```
{ athleteId, athleteName, date: 'YYYY-MM-DD',
  skills: { skating, stickhandling, passing, shooting, puckControl, hockeyIQ,
            mentalGame, sportsmanship, athleticMovement, confidence, focus,
            effortAttitude },   // each: 'beginning'|'developing'|'improving'|'consistent'|'advanced'
  notes, coachUid, coachName, createdAt }
```
Older assessments created before this skill/scale change used different
keys and a 1–5 number — the app only renders skills it recognizes by
today's key names, so pre-existing data (if any) won't show under the new
labels.

**`ah_library/{id}`** — videos & drills:
```
{ kind: 'video' | 'drill', title, category, level, videoUrl,
  description, authorUid, authorName, createdAt }
```

**`ah_announcements/{id}`** — `{ title, body, authorUid, authorName, createdAt }`

**`ah_staff/{uid}`** — coach directory for the message picker:
`{ name, email, updatedAt }` (auto-created when a coach opens the console).

**`ah_settings/{sportKey}`** — small per-sport settings doc, currently just
`{ programName, updatedAt }` (e.g. `ah_settings/hockey`). Shown on both the
Coach Dashboard hero and the athlete's Player Portfolio hero; a coach edits
it by clicking the program-name line on the dashboard.

**`ah_threads/{uidA_uidB}`** — one DM thread per user pair (id is both uids
sorted and joined with `_`, which prevents duplicate threads):
```
{ participantUids: [uidA, uidB], participants: {uid: name},
  lastMessage, lastSenderUid, createdAt, updatedAt }
```
with subcollection **`messages/{id}`** —
`{ senderUid, senderName, text, createdAt }`.

## Required setup: Firestore rules

Add the following in the
[Firebase console](https://console.firebase.google.com/project/cxa-parent-portal/firestore/rules)
**alongside** your existing parent-portal and Learn to Skate rules (rules
are additive — an operation is allowed if *any* matching rule allows it, so
adding a second `match /users/{uid}` block is fine):

```
function ahSignedIn() {
  return request.auth != null;
}
function ahMe() {
  return get(/databases/$(database)/documents/users/$(request.auth.uid)).data;
}
function ahIsStaff() {
  return ahSignedIn() &&
    exists(/databases/$(database)/documents/users/$(request.auth.uid)) &&
    ahMe().role in ['coach', 'admin'];
}

// Athlete Hub self-registration (athletes & parents). Order matters in the
// client: it links the ah_athletes doc FIRST, then creates the users doc —
// so these rules can verify the link instead of trusting the request.
match /users/{uid} {
  allow get: if ahSignedIn() && (request.auth.uid == uid || ahIsStaff());
  allow list: if ahIsStaff();
  allow create: if request.auth.uid == uid && (
    (
      request.resource.data.role == 'athlete' &&
      request.resource.data.keys().hasOnly(['name', 'email', 'role', 'athleteId', 'createdAt']) &&
      request.resource.data.name is string && request.resource.data.name.size() > 0 &&
      request.resource.data.athleteId is string &&
      get(/databases/$(database)/documents/ah_athletes/$(request.resource.data.athleteId)).data.uid == request.auth.uid
    ) || (
      request.resource.data.role == 'parent' &&
      request.resource.data.keys().hasOnly(['name', 'email', 'role', 'athleteIds', 'createdAt']) &&
      request.resource.data.name is string && request.resource.data.name.size() > 0 &&
      request.resource.data.athleteIds is list && request.resource.data.athleteIds.size() == 1 &&
      request.auth.uid in get(/databases/$(database)/documents/ah_athletes/$(request.resource.data.athleteIds[0])).data.parentUids
    )
  );
  allow update, delete: if ahIsStaff();
}

match /ah_athletes/{athleteId} {
  // 'get' (single doc by code) is enough for signup + showing the athlete's
  // own profile; browsing the whole roster ('list') stays staff-only.
  allow get: if ahSignedIn();
  allow list, create, delete: if ahIsStaff();
  allow update: if ahIsStaff() || (
    // An athlete claiming their code: only flips uid from null to their own.
    ahSignedIn() &&
    request.resource.data.diff(resource.data).affectedKeys().hasOnly(['uid', 'updatedAt']) &&
    resource.data.uid == null &&
    request.resource.data.uid == request.auth.uid
  ) || (
    // A parent linking to the athlete: only appends their own uid.
    ahSignedIn() &&
    request.resource.data.diff(resource.data).affectedKeys().hasOnly(['parentUids', 'updatedAt']) &&
    request.resource.data.parentUids is list &&
    request.resource.data.parentUids.hasAll(resource.data.parentUids) &&
    request.auth.uid in request.resource.data.parentUids &&
    request.resource.data.parentUids.size() <= resource.data.parentUids.size() + 1
  );
}

match /ah_posts/{id} {
  allow read: if ahSignedIn();
  allow create, update, delete: if ahIsStaff();
}
match /ah_announcements/{id} {
  allow read: if ahSignedIn();
  allow create, update, delete: if ahIsStaff();
}
match /ah_library/{id} {
  allow read: if ahSignedIn();
  allow create, update, delete: if ahIsStaff();
}
match /ah_staff/{uid} {
  allow read: if ahSignedIn();
  allow write: if ahIsStaff();
}
match /ah_settings/{sportKey} {
  allow read: if ahSignedIn();
  allow write: if ahIsStaff();
}

match /ah_assessments/{id} {
  // Staff, the assessed athlete, and that athlete's parents.
  allow read: if ahIsStaff() || (
    ahSignedIn() && (
      resource.data.athleteId == ahMe().get('athleteId', '') ||
      resource.data.athleteId in ahMe().get('athleteIds', [])
    )
  );
  allow create, update, delete: if ahIsStaff();
}

match /ah_threads/{threadId} {
  allow read: if ahSignedIn() && request.auth.uid in resource.data.participantUids;
  allow create: if ahSignedIn() &&
    request.auth.uid in request.resource.data.participantUids &&
    request.resource.data.participantUids.size() == 2 &&
    request.resource.data.keys().hasOnly(['participantUids', 'participants', 'lastMessage', 'lastSenderUid', 'createdAt', 'updatedAt']);
  allow update: if ahSignedIn() &&
    request.auth.uid in resource.data.participantUids &&
    request.resource.data.diff(resource.data).affectedKeys().hasOnly(['lastMessage', 'lastSenderUid', 'updatedAt', 'participants']);
  allow delete: if ahIsStaff();

  match /messages/{messageId} {
    allow read: if ahSignedIn() &&
      request.auth.uid in get(/databases/$(database)/documents/ah_threads/$(threadId)).data.participantUids;
    allow create: if ahSignedIn() &&
      request.auth.uid in get(/databases/$(database)/documents/ah_threads/$(threadId)).data.participantUids &&
      request.resource.data.senderUid == request.auth.uid &&
      request.resource.data.keys().hasOnly(['senderUid', 'senderName', 'text', 'createdAt']) &&
      request.resource.data.text is string &&
      request.resource.data.text.size() > 0 && request.resource.data.text.size() <= 2000;
    allow update, delete: if false;
  }
}
```

Also make sure **Email/Password** sign-in is enabled in Firebase Auth
(it already is if the parent portal uses it).

No composite indexes are needed — every query is a single-field filter and
sorting happens in the browser.

**Security notes (same trust model as Learn to Skate):**

- Anyone who knows an *unclaimed* athlete code could create the athlete
  login for it, and anyone who knows *any* code could link themselves as a
  parent. Codes are random 6-character strings that are only shared
  directly with families, and coaches can see exactly which accounts are
  linked to each athlete (and unlink them) from the roster. If you want
  stronger guarantees later, Firebase App Check or a Cloud Function
  approval step are the upgrades to consider.
- Direct-message threads are private to their two participants — other
  coaches can't read them. Keep that in mind for club policies about
  adult/minor communication; a common practice is to message the parent,
  or put general feedback in the (coach-visible) training feed instead.
- Posts, announcements, and the library are readable by **any signed-in
  hub account** (the app filters the feed per athlete for tidiness, not
  secrecy). Skill assessments are locked down: only staff, the athlete,
  and their linked parents can read them.

## Deployment

The folder deploys with the rest of the repo via GitHub Pages and is live
at:

- Athlete Hub: https://cross0816.github.io/Carlos/athlete-hub/
- Coach Console: https://cross0816.github.io/Carlos/athlete-hub/coach.html

(Note: `lts.coloradoextreme.org` is a **separate** Pages site that serves
only the Learn to Skate app, so the hub is not reachable there.) All
internal links are relative, so the app also works unchanged behind a
custom domain (e.g. a future `hub.coloradoextreme.org` pointed at this
repo's Pages site) — the hub would then be at
`hub.coloradoextreme.org/athlete-hub/`.
