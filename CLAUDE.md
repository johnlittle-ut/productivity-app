# CLAUDE.md — Productivity App

This file is read by **both** Claude chats working on this project. Read it fully before doing anything.

## Roles

### John (the user)
- Owner of the project and the go-between for the two chats.
- Plans with Craig 2, copies Craig 2's handoffs to Dan 2, and tests the app on his iPhone and Windows laptop.
- Develops from a **Windows laptop**. There is **no Mac**. Only Apple device is an iPhone.

### Craig 2 (planning chat)
- **Planning, architecture, and requirements only. Craig 2 NEVER writes application code.**
- Allowed outputs: `.md` files, planning documents, and handoff instructions for Dan 2.
- Handoffs to Dan 2 are given **inline in chat as copy-paste text** so John can paste them straight into Dan 2's chat.
  - Only create an actual handoff file (e.g. `handoffs/001-initial-build.md`) when it is large. The first handoff (initial build spec) is one of these.
- Keeps this CLAUDE.md up to date when roles, workflow, or key decisions change.

### Dan 2 (coding chat)
- **Implements everything.** Writes, edits, and tests all code.
- Works primarily from handoffs Craig 2 has produced and John has pasted in. If a handoff is ambiguous or conflicts with this file, Dan 2 asks John (who can ask Craig 2) instead of guessing. John will occassionally request changes directly. this is totally fine as well but will usually be for small things
- **After every set of changes, Dan 2 must update `CHANGELOG.md`** (see below) before reporting back as done.
- Does not make architecture or scope decisions on his own. Flag them to John instead.

## Workflow

1. John and Craig 2 plan a feature or change.
2. Craig 2 writes the handoff (inline copy-paste, or a file if large).
3. John pastes the handoff to Dan 2.
4. Dan 2 makes the changes.
5. Dan 2 updates `CHANGELOG.md`.
6. John tests, then returns to Craig 2 with results or the next idea.

## Changelog rules (for Dan 2)

- File: `CHANGELOG.md` in the project root. Create it on the first build.
- Newest entry on top. Each entry has: date, short title, handoff reference, and bullet list of what changed (files touched, behavior added or changed, anything left undone or any deviation from the handoff).
- Never skip an entry, even for small changes.

## Project summary

A personal calendar / to-do app for John's iPhone (also usable on a Windows laptop). It may later be shared, invite-only, with a girlfriend and/or roommates, each with their own private data. John treats his calendar like a to-do list. His current system is one long OneNote page listing every date of the year (Mondays in bold) with homework due dates, regular events, and special events. The app replaces this.

**Core pain point:** regular calendar apps send excessive notifications (e.g. 15 for one event), which also clutter his CarPlay display. Notifications in this app must be minimal, opt-in per item, and one clean notification per item.

### Planned tabs
1. **Calendar** — add and view events.
2. **Daily Schedule** — auto-fills that day's events from the Calendar in a layout similar to the OneNote setup, plus per-day to-do items.
3. **Shopping List**
4. **General To-Do** — not tied to a date.

## Decisions made so far

| Topic | Decision |
|---|---|
| Platform | **PWA** (web app installed to iPhone Home Screen). No App Store, no Apple Developer account. |
| Hosting | **Firebase Hosting (free)** preferred, so Google sign-in works reliably in the iPhone Home Screen app. Code in a private GitHub repo (John has an account). GitHub Pages is the fallback. |
| Cost | **$0 recurring.** A very small one-time cost is acceptable in the worst case. No ongoing hosting fees. |
| Devices | **iPhone is the primary target.** The Windows laptop is **kept as a supported, synced secondary** (same PWA codebase opened in a browser, not a separate app). Laptop must work and look reasonable, but gets **no dedicated polish pass**. Confirmed by John 2026-11-03. |
| Auth and access | **Google sign-in, invite-only, multiple users (decided 2026-11-03, replaces "locked to John only").** Option A: an **allow-list in Firestore**, documents `allowedEmails/{lowercased email}`, managed by John in the Firebase console (no rules redeploy needed to add someone). Rules let a user touch only their own `users/{uid}/...` and only while on the allow-list. Not-invited users see a friendly "invite-only" screen, never a raw error. **Never hard-code a UID or email.** Details in `Plan.md` section 3.0. |
| Per-user everything | Each person has their own private calendar, lists and **settings (timezone, digest time, etc.)**. Phase 2 push subscriptions and Phase 3 widget tokens are also **per-user**; one Cloudflare Worker serves all users. |
| Shared calendar | **Phase 4 (final), rough idea only, nothing designed yet.** Only preparation now: put all item reads and writes behind one data-layer module, and do not add a `calendarId` field. See `Plan.md` Phase 4. |
| Data safety | Local-first storage (IndexedDB) + cloud sync + manual JSON export/import backup. Must not lose the calendar if something glitches. |
| Sync backend | **Firebase** (Firestore + Google sign-in), free Spark tier. John has **never used Firebase**, so Dan 2 must give clear, step-by-step, Windows-friendly setup instructions (create project, enable Google sign-in, set Firestore security rules locking data to John's account, etc.) and not assume prior knowledge. Firestore storage is separate from John's Google One storage plan, so it does not use his photo/email space. |
| Item model | One unified "item" with a **type of Event or Task** (deadlines like "rent due" or "Module 2 due at 11:59" are Tasks with a date/time). Fields: title, type, date, optional time, optional notes, done flag, optional repeat, optional notification. Events and tasks use **different colors**. No other categories/tags. |
| Views | Calendar tab and Daily Schedule tab show the **same underlying data** differently. **Calendar tab = week view.** **Daily Schedule tab = styled like John's OneNote page** (date headers, items listed beneath, weeks start on Monday and are visually marked, timed items in order, untimed tasks as a checklist). |
| Shopping List | **One single list**, one tab, no separation by store. |
| Repeats | **Weekly and monthly only.** |
| Completion | Check-off hides the item, and **completed items are auto-cleared 24 hours after being checked off.** Dates are generated, never stored, so there are no past dates to delete. Manual delete is still available. |
| Data sensitivity | Not sensitive (no passwords, financial, etc.). |
| Notifications | Regular iPhone notifications (NOT calendar-style spam). **One notification per item.** Adjustable per item; **default for Events is 1 hour before.** Never ask for permission unprompted. |
| Daily digest | **One notification at 9:00 AM daily** summarizing that day's calendar and to-do items. (Delivery mechanism TBD, see open questions.) |
| Widget | John wanted an interactive widget, but that is **not possible with a PWA**. **Decision: read-only widget via the free Scriptable app, built in Phase 3.** |
| Master plan | **`Plan.md`** is the detailed plan and source of truth for scope, data model, screens, and phases. Read it before starting any work. If a handoff conflicts with it, ask John. |

## Open questions (not yet decided)

- **Notification delivery:** iOS PWAs support Web Push (iOS 16.4+, only when installed to Home Screen). Scheduled pushes need a server-side sender. Must find a **$0** way to do this (e.g. Cloudflare Workers cron triggers, or Firebase options). Firebase Cloud Functions requires a billing-enabled plan, so check that it stays within free limits and acceptable to John before choosing.
- **Home screen widget:** a PWA cannot have one. Options being discussed: (a) read-only widget via the free Scriptable app, (b) skip the widget, (c) native app later (needs Apple Developer account, $99/yr, which breaks the $0 rule). Interactive check-off from a widget is not achievable for free.
- **9:00 AM digest and 1-hour-before reminders:** both need a server-side sender (or an iOS Shortcuts automation as a fallback for the digest). Timezone: John appears to be in Eastern time; scheduling must handle daylight saving.
- **Build phases:** Phase 1 = four tabs, Google sign-in with invite-only allow-list, Firebase sync, backup export. Phase 2 = notifications and digest (per-user). Phase 3 = read-only widget (per-user token) and polish. **Phase 4 = shared calendar (rough idea, not designed).**
- **Possible Workspace issue:** a company or school Google account may be blocked by its admin from signing in to outside apps. Use personal Google accounts, and test when the first extra person is invited.

## Constraints for Dan 2

- Develop and test on **Windows**. Do not require a Mac or Xcode.
- Prefer simple, maintainable tech. Avoid anything that needs paid services.
- The app must work offline and sync when back online.
- Keep the UI mobile-first (iPhone), but usable on a laptop browser. The laptop is the same PWA in a browser; do not build or maintain a separate desktop app. Do not spend extra time polishing laptop layouts beyond making them work and look reasonable.
- Do not add notifications, tracking, or analytics that John has not asked for.
- Build everything **per-user** from the start: data under `users/{uid}`, settings per user, no global user-specific constants (no hard-coded UID, email, or timezone).
