# Plan.md — Personal Calendar / To-Do App

Status: **Planning complete for Phase 1. Living document, expect tweaks.**
Owner: John. Planner: Craig 2. Builder: Dan 2. (Roles and workflow are in `CLAUDE.md`.)
Last updated: 2026-11-03 (timezone, overdue section, and list ordering confirmed; **multi-user invite-only access and a Phase 4 shared calendar added**)

---

## 1. Goal and problem

John uses a single long OneNote page as a calendar and to-do list: every date of the year, Mondays in bold, with homework due dates, bills, regular events and special events typed underneath. When something is done he deletes it, and later deletes the past dates.

Regular calendar apps are unusable for him because they send far too many notifications (e.g. 15 for one event) and the alerts block his CarPlay display.

**Goal:** a personal app, used on his iPhone and Windows laptop, that behaves like his OneNote page but with automatic repeats, check-off, clean cleanup, and **minimal, deliberate notifications.**

**Sharing (added 2026-11-03):** John may later let a girlfriend and/or roommates use the app. Each person signs in with their own Google account and gets their **own private calendar, lists and settings.** A **shared calendar** for coordinating with one or more of them is a rough idea for the final phase (Phase 4). The app is **invite-only**, not open to the public.

### Hard constraints
- **$0 recurring cost.** (A very small one-time cost is acceptable only as a last resort.)
- **Developed on Windows.** No Mac, no Xcode, no Apple Developer account.
- **Invite-only** (Google sign-in plus an allow-list John controls). Each person's data is private to them. Not sensitive data, but must not be lost.
- **Works offline,** syncs when back online.
- **Notifications are opt-in per item and one-per-item.** No unprompted permission requests, no tracking, no analytics.

---

## 2. Architecture decisions

| Area | Decision | Notes |
|---|---|---|
| App type | **PWA** installed to iPhone Home Screen (Safari, Share, Add to Home Screen) | Also works in a laptop browser. |
| Frontend | **Vite + React + TypeScript**, `vite-plugin-pwa` for manifest and service worker | Dan 2 may propose Preact or similar if lighter, but must flag it to John first. |
| Backend / sync | **Firebase (Spark free plan):** Authentication (Google provider) + Firestore | John has never used Firebase. Dan 2 must give step-by-step setup instructions. |
| Local-first | Firestore **offline persistence** (IndexedDB) | The app reads and writes locally and syncs automatically. |
| Hosting | **Firebase Hosting (free)** preferred. Source code lives in a private GitHub repo. | Reason: Google sign-in via redirect can break in iOS Home Screen apps when the auth domain differs from the site domain. Hosting on Firebase keeps them the same. Dan 2 must **test sign-in on the real iPhone early** (see Risks). GitHub Pages is the fallback. |
| Backup | Manual **Export / Import JSON** in Settings | Insurance against any free-tier or account problem. |
| Access control | **Invite-only allow-list (decided 2026-11-03, option A).** Allowed people are stored as documents `allowedEmails/{lowercased email}` in Firestore. Rules admit a signed-in user only if their verified email has a document there. John adds or removes people in the Firebase console. | No rules edit or redeploy needed to add a person. Open sign-in (any Google account) was rejected. See section 3.0. |
| Security | Each person can read and write **only their own** `users/{uid}/...` data, and only while on the allow-list. Everything else denied. | Never hard-code a UID or email in app code or rules. |
| Timezone | **Per-user setting.** John's is **America/New_York** (Louisville). New users default to their browser's timezone and can change it in Settings. | Dates stored as plain local `YYYY-MM-DD` strings, times as `HH:mm`, so DST never shifts an event. |
| Notifications (Phase 2) | Web Push to the installed PWA, sent by a free **Cloudflare Worker** on a cron timer | See section 7. |
| Widget (Phase 3) | **Read-only** widget via the free **Scriptable** iOS app | Decided: option A. See section 8. |

### Why a PWA and not a native app
iOS widgets and App Store apps require Apple's $99/year program. PWAs are free and need no Mac. The tradeoff is that interactive widgets are impossible. John accepted this.

---

## 3. Data model (Firestore)

All personal data lives under the signed-in user: `users/{uid}/...` (so every person is already separated). The sections below describe the shape for **one user**; every user gets their own copy.

### 3.0 Access control and multi-user

**Allow-list.** Top-level collection `allowedEmails`. Each document ID is the person's **lowercased Google email**; fields are just `addedAt` and an optional `note` (e.g. "girlfriend"). John creates and deletes these documents by hand in the Firebase console (the console bypasses rules). Using the email as the key means John can invite someone **before** they have ever signed in (their UID is not known until they do).

**Rules summary (Dan 2 writes the actual rules):**
- `users/{uid}/**`: allowed only if the user is signed in, `request.auth.uid == uid`, the email is verified, and `allowedEmails/{their lowercased email}` exists.
- `allowedEmails/{email}`: a signed-in user may `get` **only their own** document (so the app can tell "invited" from "not invited" and show a friendly message). No listing, and **no client writes at all.**
- Everything else: denied.
- Note: the rule's lookup counts as a small Firestore read. That is negligible at this scale (Spark allows 50,000 reads per day), but keep the app from doing needless repeated reads.

**First-time sign-in for an invited user:** the app creates their `settings` document with defaults (timezone from the browser, default reminder 60 minutes, digest 9:00 AM, week starts Monday). Their lists and calendar start empty.

**Not-invited users:** they can complete Google sign-in (Firebase Auth will create an account record), but they see a friendly **"This app is invite-only"** screen showing the email they signed in with, plus text like "Ask John to add this email," and a Sign out button. No raw error codes.

**Reserved for the Phase 4 shared calendar (decision):** do **not** add a `calendarId` field to items now. Personal items live under `users/{uid}/items`. Shared calendar items will later live in a separate collection such as `calendars/{calendarId}/items` with the **same item shape**, and a `calendars/{calendarId}` document holding a member list. Where an item lives tells you which calendar it belongs to, so no migration is needed. The one thing to do now (handoff 002): put **all item reads and writes behind one data-layer module**, so a second data source (a shared calendar) can be merged into the views later without rewriting the screens.

### 3.1 `items` (events and tasks)
One shared collection. Calendar and Daily Schedule are just two views of it.

| Field | Type | Notes |
|---|---|---|
| `id` | string | Auto-generated. |
| `title` | string | Required. |
| `type` | `"event"` or `"task"` | Deadlines (rent due, assignment due 11:59) are **tasks** with a date and time. |
| `date` | string `YYYY-MM-DD` | Required. For repeating items this is the first occurrence. |
| `startTime` | string `HH:mm` or null | Null means all-day or untimed. |
| `endTime` | string `HH:mm` or null | Optional, events only. |
| `notes` | string | Optional longer text (e.g. "pregame at my apartment"). |
| `done` | boolean | Non-repeating items only. |
| `doneAt` | timestamp or null | Used for the 24-hour auto-clear. |
| `repeat` | null or `{ freq: "weekly" \| "monthly" }` | Weekly and monthly only. Optional `until` date is allowed. |
| `completedOccurrences` | map `{ "YYYY-MM-DD": doneAtTimestamp }` | Repeating items only. Check-off applies to one occurrence. |
| `skippedDates` | array of `YYYY-MM-DD` | Repeating items only. "Delete just this one." |
| `reminder` | `{ enabled: bool, minutesBefore: number }` | Default for events: enabled, 60. Default for tasks: off. Adjustable per item. |
| `createdAt`, `updatedAt` | timestamp | |

**Recurrence rules**
- Weekly: same weekday as `date`. Monthly: same day-of-month as `date`; if the month is shorter (e.g. the 31st), use the **last day** of that month.
- Occurrences are **calculated by the app, never stored.** This is also why there are no "past dates to delete."
- Editing or deleting a repeating item asks: **this occurrence only** or **all occurrences.** (Do not build "this and following" for now.)

### 3.2 `shoppingItems`
`id`, `title`, `done`, `doneAt`, `createdAt`, `order` (number, for manual ordering). One single list.

### 3.3 `todos` (general, not date-specific)
`id`, `title`, `done`, `doneAt`, `createdAt`, `order` (number, for manual ordering). Optional `notes`.

**Ordering rule (both lists):** new items are added at the **bottom** (highest `order`). John can **drag to reorder** any item, which rewrites its `order`. Use a gap-based or fractional order value so one drag only updates one document.

### 3.4 `settings` (single document, **per user**)
`timezone`, `defaultEventReminderMinutes` (60), `digestEnabled` (true), `digestTime` ("09:00"), `weekStartsOn` ("monday"), and later that user's push subscription(s) and widget token. **Nothing in settings is global**; each person has their own.

### 3.5 Completion and cleanup
- Checking an item off sets `done: true` and `doneAt: now` (or records the occurrence in `completedOccurrences`).
- Done items **disappear from normal views right away** and are **permanently deleted 24 hours after `doneAt`.**
- Cleanup runs on the client when the app opens and when it returns to the foreground. (No server is needed for this in Phase 1.)
- Within the 24 hours, an **"Undo"** is available (a small "Recently completed" area or a toast) so an accidental tap is recoverable.
- Manual delete is always available.

---

## 4. Screens and behavior

Bottom tab bar with four tabs, plus a Settings screen reached by a gear icon.

### Global design
- **Mobile-first.** iPhone is the primary target. The laptop is the **same PWA in a browser** (one codebase), fully usable with wider layouts, but with **no dedicated polish pass**. Confirmed by John: keep laptop support, lower priority.
- Follows system light and dark mode.
- **Colors:** events and tasks have clearly different colors (e.g. events blue, tasks orange). Also use a different icon or shape, not color alone, so it stays readable.
- No other categories or tags.
- A floating **"+" button** on Calendar and Daily Schedule opens the Add/Edit form.

### 4.1 Add / Edit item form
Fields: Title, Type (Event or Task toggle), Date, Time (optional), End time (optional, events), Notes, Repeat (None / Weekly / Monthly), Reminder (Off / At time / 15 min / 30 min / 1 hour / 2 hours / morning of / custom). Event default reminder is 1 hour before.

Note: in Phase 1 the reminder fields are **saved but do nothing yet** (notifications arrive in Phase 2). The UI may show them with a small "coming soon" note, or hide them until Phase 2. **Dan 2: hide them until Phase 2, but keep the fields in the data model.**

### 4.2 Calendar tab (week view)
- Shows one week at a time, **Monday through Sunday**, with previous / next / "Today" controls. Current day is highlighted.
- **Laptop:** seven day columns. Timed items placed in order, untimed items at the top of each column.
- **Phone:** a compact Monday-Sunday strip at the top (dots show days with items), with the selected day's items listed below. Swipe left or right to change week. *(This is Craig 2's proposal for a narrow screen. John may tweak it after seeing it.)*
- Tap an item to view, edit, check off or delete. Tap an empty area or "+" to add on that day.

### 4.3 Daily Schedule tab (OneNote style)
Modeled on John's OneNote page:
- A **continuous scrolling list from today forward**, one block per date, with a header like `11/2` (and day name).
- **Mondays are bold,** with extra spacing, replacing the bolded Mondays in OneNote.
- **Every date is shown, even empty ones** (like OneNote), so it is easy to add something to any day. Load further dates as John scrolls (e.g. 60 days at a time).
- Under each date: **timed items in time order first,** then untimed items as a checklist. Item text shows the time like OneNote ("Thrive at 6:30").
- Each item has a check circle. Checking it off hides it (see 3.5).
- A quick inline "add" on each date (title only, then expand for more details).
- **Overdue section (proposed):** unfinished tasks from past dates appear in an "Overdue" block at the top, so nothing silently disappears the way deleted dates do in OneNote. Events from the past simply drop off. *John to confirm.*
- Past dates are not shown.

### 4.4 Shopping List tab
One list. Type to add. Tap to check off. Checked items are cleared after 24 hours. **New items go at the bottom.** John can **drag to reorder** (use a drag handle so scrolling on a phone does not accidentally move items; it must work with touch on iPhone and mouse on laptop).

### 4.5 General To-Do tab
Same behavior as the shopping list, for non-dated tasks: new items at the bottom, drag to reorder, check off, 24-hour clear. Optional notes.

### 4.6 Settings
Per-user: timezone, default reminder, Export JSON, Import JSON (each person exports only their own data), Sign out, and the signed-in email shown. (Digest time and notification toggles added in Phase 2.)

### 4.7 Sign-in screen
"Sign in with Google" button only. After sign-in, if the account's email is not on the allow-list, show the friendly **"This app is invite-only"** screen described in 3.0 (with the email shown and a Sign out button). Firestore rules enforce this on the server too, so the screen is a courtesy, not the security.

---

## 5. Build phases

### Phase 1: Core app (target: working app John can use daily)
Handoffs (each small enough for Dan 2 to finish and for John to test):

| # | Handoff | Contents |
|---|---|---|
| 001 | **Project setup and Firebase** (large file) | Repo scaffold (Vite, React, TS, PWA plugin), Firebase project and Hosting setup walkthrough, Google sign-in, **invite-only allow-list and per-user Firestore rules**, friendly "invite-only" screen, tab shell, deploy to a live URL, install on iPhone. Includes the early iPhone sign-in test. Also creates `CHANGELOG.md`. |
| 002 | **Data layer and Add/Edit form** | Types, Firestore reads and writes with offline persistence, recurrence engine (weekly, monthly, month-end clamp), add / edit / delete / this-or-all prompt. |
| 003 | **Daily Schedule tab** | OneNote-style list, Monday emphasis, check-off, undo, 24-hour cleanup, overdue section. |
| 004 | **Calendar tab (week view)** | Phone and laptop layouts. |
| 005 | **Shopping List and General To-Do** | Both lists, new items at bottom, drag-to-reorder, check-off and cleanup. |
| 006 | **Settings, backup and polish** | Export / Import JSON, icons, offline behavior check, error states. |

**Phase 1 is done when:** John can sign in on both devices, add events and tasks (including repeating ones), see them in both views, check them off, have them auto-clear after 24 hours, work offline, and export a backup. Changes made on one device appear on the other.

### Phase 2: Notifications
- Web Push set-up (service worker, subscription saved to Firestore **under that user**, permission requested **only** when the user taps "Enable notifications" in Settings).
- **Per-item reminders** (default for events: 1 hour before; adjustable).
- **Daily digest at 9:00 AM** (each user's own digest time and timezone) with the day's events and to-dos.
- **Everything is per-user:** each person has their own push subscription(s), settings, and notifications. One server job serves all allowed users.
- Server side: see section 7.

### Phase 3: Widget and polish
- Read-only home screen widget (section 8), with a **separate secret token per user**.
- Optional quick-add text parsing ("Thrive fri 6:30pm").
- Small visual tweaks, anything John wants after real use.

### Phase 4: Shared calendar (final phase, rough idea only)
**Nothing is designed yet.** Added 2026-11-03 so earlier decisions do not block it. Idea: John can coordinate with a roommate, girlfriend or close friend on a shared calendar, shown alongside his own items.

Notes to keep in mind (to be decided when this phase starts, not now):
- Needs a structure **outside** `users/{uid}`, e.g. `calendars/{calendarId}` with a member list and `calendars/{calendarId}/items`, plus rules for who can read and write.
- Members must already be on the allow-list.
- Questions for later: who can edit versus only view, whether shared items can have per-person reminders, how shared items are colored, how check-off works on a shared item, and what happens when a member leaves.
- The data-layer decision in section 3.0 is the only preparation needed now.

---

## 6. Firebase setup notes for Dan 2 (John is new to it)

Dan 2 must walk John through, one small step at a time, with exact clicks and without assuming knowledge:
1. Creating a Firebase project on the **Spark (free)** plan.
2. Enabling Google as a sign-in provider.
3. Creating the Firestore database.
4. Registering the web app and copying the config into the project (config values are not secret, but do not commit service-account keys).
5. Installing Firebase CLI on Windows and `firebase deploy`.
6. Setting up the **invite-only allow-list** (section 3.0): create `allowedEmails/{John's lowercased email}` in the console **first**, then deploy the per-user rules. (If the rules go live before John is on the list, he is locked out.) Also teach John how to add and remove another person later. John should use a **personal** Google account for the Firebase project and for signing in, not a work or school account.
7. Setting a **budget alert** (or confirming the project stays on Spark) so John can never be billed by surprise.

---

## 7. Phase 2 server design (notifications)

**Problem:** iPhone web apps can only receive a notification at a set time if a server sends the push at that time.

**Leading design:** a **Cloudflare Worker** (free tier) with a cron trigger running every 5 minutes.
1. Worker reads Firestore (REST API with a service account kept as a Worker secret) for **each allowed user's** items with reminders due in the next window, and for each user's 9:00 AM digest.
2. Worker sends **Web Push (VAPID)** to **that user's** saved subscription(s). It never sends one person's items to another person.
3. Worker marks reminders as sent (so there is exactly **one** notification per item, never duplicates).
4. Each digest check is done in **that user's own timezone and digest time** so daylight saving does not break it.
5. A single Worker config serves everyone; there is no per-user deployment. Adding a person means adding their email to the allow-list, and they enable notifications on their own phone.

**Alternative:** Firebase scheduled Cloud Functions. Rejected for now because it requires the Blaze (pay-as-you-go) plan with a credit card on file. It would likely stay free at this scale, but John prefers to avoid it. Revisit only if the Worker approach fails.

**Fallback for the digest:** an iOS **Shortcuts personal automation** at 9:00 AM that fetches the day's summary and shows a normal notification. This needs no push at all.

**Known risks**
- Web push on iPhone only works when the app is **installed to the Home Screen** (iOS 16.4 or later).
- iOS web push is somewhat less reliable than native push. Dan 2 should test over several days before John depends on it.
- Notifications should be configured so they are standard single notifications and do not repeat, to protect CarPlay.

---

## 8. Phase 3: widget design (Scriptable)

- **Read-only.** iOS does not allow interactive widgets for PWAs. John cannot check off items from the widget. Tapping it opens the PWA.
- John installs the free **Scriptable** app. Dan 2 writes the widget script (JavaScript) and John pastes it in.
- The script fetches today's events and tasks from a small endpoint (the same Cloudflare Worker) protected by a long **secret token** in the URL. Not sensitive data, but the URL must not be guessable. **Each user has their own token**, which maps to that user only (e.g. a `widgetTokens/{token}` lookup to a UID, written by the app on the user's behalf).
- Shows today's date, timed items in order, then tasks, with the event and task colors.
- Refresh time is controlled by iOS and is not guaranteed to be instant. Set expectations accordingly.

---

## 9. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Google sign-in fails in iOS Home Screen mode | Host on Firebase Hosting (same domain for auth). **Test on the real iPhone in handoff 001** before building anything else. |
| iOS clears web storage | Firestore holds the real copy, plus JSON export. |
| Push notifications are unreliable | Test early in Phase 2. Fall back to the Shortcuts digest. |
| Free-tier changes or account problems | Regular JSON export. Data is small and portable. |
| Accidental deletion or check-off | 24-hour undo window, confirm prompt on delete. |
| Scope creep | New ideas go on the list in section 11, not straight into the build. |
| An invited person's Google account is a work or school (Workspace) account that the admin blocks from outside apps | Prefer personal Google accounts for everyone. Test when the first person is invited. John's own Firebase project and sign-in should also be a personal account. |
| Someone not invited signs in anyway | Rules deny all data; app shows the friendly invite-only screen. They may appear in Firebase Authentication's user list, which is harmless and can be deleted there. |
| John removes someone from the allow-list | They are blocked immediately; their old data remains under their UID until John deletes it in the console (document this in the README). |

---

## 10. Out of scope (for now)
- Interactive widget, native iOS app, App Store.
- Open public sign-up. (Several invited users is **in scope** from Phase 1. A shared calendar is **Phase 4**, not earlier.)
- An in-app admin screen for managing the allow-list (John uses the Firebase console; revisit later if it gets annoying).
- Tags, categories, or source labels beyond event vs task color.
- Recurrence other than weekly and monthly.
- Syncing with Apple Calendar or Google Calendar.
- Analytics of any kind.

---

## 11. Open questions and tweak list
- [x] Timezone is **America/New_York**. (Confirmed by John.)
- [x] **Overdue** section for unfinished past tasks. (Confirmed by John.)
- [ ] Confirm the phone layout proposal for the Calendar week view (strip plus selected-day list) after seeing it.
- [x] New shopping and to-do items go at the **bottom**, with drag-to-reorder. (Confirmed by John.)
- [x] Access model: **invite-only allow-list keyed by email** (option A). (Decided by Craig 2, 2026-11-03.)
- [ ] John's personal Google email to seed the allow-list (collected during handoff 001).
- [ ] Phase 4 shared calendar design (later).
- [ ] Where Dan 2's code workspace lives on John's laptop (recommended: outside OneDrive, see handoff 001, step 0).
- [ ] Items John wants later: ___

---

## 12. Next action
Craig 2 writes **handoff 001** as `handoffs/001-project-setup-and-firebase.md` (large, so it is a file). John passes it to Dan 2.
