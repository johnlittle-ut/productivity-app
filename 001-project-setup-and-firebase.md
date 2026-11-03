# Handoff 001: Project setup, Firebase, sign-in, and deploy

From: Craig 2 (planner). To: Dan 2 (builder). Relayed by John.
Phase: 1 (Core app). Date: 2026-11-03.

## REVISION 2026-11-03: access model changed (read this first)
The app is now **invite-only for several Google accounts**, each with its own private data. This **replaces** the earlier "locked to John's single UID" approach. Decision: **option A, an allow-list keyed by email, stored in Firestore.** **Step 4, Step 5, the step 7 checklist, the deliverables, and the definition of done below are updated to match.** Steps 0 to 3 and 6 are unchanged. Full details are in `Plan.md` section 3.0. Do not hard-code any UID or email in app code or rules.

## Read first
1. `CLAUDE.md` (roles, workflow, changelog rules).
2. `Plan.md` (the full plan, source of truth). This handoff implements only the parts described below.

Remember: John has **never used Firebase** and is on **Windows with no Mac**. Explain every console step with exact clicks, one small step at a time, and wait for John to confirm each step before moving on. Do not dump a 30-step wall of text.

## Goal
End of this handoff: John can open a **live URL** on his iPhone, install it to the Home Screen, tap **Sign in with Google**, and land on an empty four-tab app shell. A Google account that is **not on the invite list** gets a friendly "invite-only" screen and no data access. An invited account sees only its own private data. Nothing else (no items, no calendar logic) is built yet.

The most important outcome is proving that **Google sign-in works inside the iPhone Home Screen app.** If it does not, stop and report to John before building anything further (see step 7).

## Step 0: Working location (ask John first)
The planning docs live in John's OneDrive folder `...\Pictures\Screenshots\Productivity App`. A `node_modules` folder inside OneDrive will sync tens of thousands of files and cause problems.
- **Recommend:** create the code project **outside OneDrive**, e.g. `C:\dev\productivity-app`.
- Ask John to confirm the location, and how Dan 2 should keep `CLAUDE.md`, `Plan.md`, `CHANGELOG.md` and `handoffs/` available in the project. Suggested: copy them into the project root and tell John that Craig 2's copies in OneDrive are the masters (John re-copies when Craig 2 updates them).
- Do not proceed until John has answered.

## Step 1: Prerequisites on John's Windows laptop
Check, and help install only what is missing: **Node.js LTS**, **Git**, and the **Firebase CLI** (`npm install -g firebase-tools`). Confirm versions. John needs a **GitHub account** (he has one) and a Google account for Firebase.

## Step 2: Scaffold the project
- Vite + React + TypeScript.
- `vite-plugin-pwa` configured with a web app manifest: name "Productivity App" (placeholder, John may rename), `display: standalone`, theme and background colors, and simple placeholder icons (192, 512, and an Apple touch icon 180x180). Placeholder icons are fine for now.
- Service worker that caches the app shell so it opens offline.
- Add iOS meta tags needed for Home Screen installs (`apple-mobile-web-app-capable`, status bar style, viewport with `viewport-fit=cover`).
- `git init`, a proper `.gitignore` (node_modules, dist, .env files, Firebase debug logs), and a **private** GitHub repo. Walk John through creating the repo and pushing.

## Step 3: Firebase project (Spark free plan only)
Walk John through, in the Firebase console:
1. Create a project on the **Spark (free) plan**. Do NOT upgrade to Blaze. Disable Google Analytics when asked. John should use a **personal** Google account for this (not a work or school account), since employer admins can block outside-app sign-ins.
2. Build, Authentication, enable the **Google** provider.
3. Build, Firestore Database, create database in **production mode**, pick a US region (e.g. `nam5` or `us-east1`). (Region cannot be changed later, so tell John before he clicks.)
4. Register a **Web app** and copy the config into the project. Use Vite env variables (`VITE_FIREBASE_*`) in a local `.env` file that is git-ignored, plus a committed `.env.example` with blank values. (These web config values are not secrets, but keep the habit clean.)
5. Enable **Firebase Hosting** and link the project via the CLI (`firebase login`, `firebase init hosting`: public directory `dist`, single-page app **yes**, GitHub auto-deploy **no** for now).

## Step 4: Sign-in
- Sign-in screen: a single **"Sign in with Google"** button, nothing else.
- Use Firebase Auth with `signInWithPopup` first. If popups misbehave in the iPhone Home Screen app, fall back to `signInWithRedirect`. Set `authDomain` so that auth runs on the **same domain as the hosted app** (the Firebase Hosting domain), since cross-domain redirect auth is the known failure mode on iOS.
- Keep the user signed in across app restarts (default persistence is fine; verify it survives closing and reopening the Home Screen app).
- Add a **Sign out** button (temporary location is fine; the Settings screen comes later).
- Show a clear message while loading auth state so there is no flash of the sign-in screen for an already signed-in user.

## Step 5: Invite-only access with private per-user data
**Model (decided):** an allow-list in Firestore. Each allowed person is one document `allowedEmails/{their lowercased Google email}` with fields `addedAt` (timestamp) and optional `note` (string). John manages these documents by hand in the Firebase console, so adding a person never requires editing or redeploying rules. Keyed by email so John can invite someone before they have signed in.

1. **Seed John first.** In the Firestore console, John creates the collection `allowedEmails` and a document whose ID is his own lowercased email. **Do this BEFORE deploying the rules**, otherwise John locks himself out. Walk him through the exact clicks.
2. Write `firestore.rules`:
   - `users/{uid}/**`: read and write allowed only if the user is signed in, `request.auth.uid == uid`, `request.auth.token.email_verified == true`, and `allowedEmails/{lowercased request.auth.token.email}` exists.
   - `allowedEmails/{email}`: a signed-in user may **get only their own** document (email matches theirs). **No list. No client writes.**
   - Everything else: deny.
   - No UID or email hard-coded anywhere in the rules. Deploy with `firebase deploy --only firestore:rules`.
3. **In the app,** after sign-in, check whether the user's own `allowedEmails` document exists:
   - **Yes:** continue to the app shell, and create that user's `users/{uid}/settings` document if it does not exist yet (timezone from the browser, default reminder 60 minutes, digest enabled at "09:00", week starts Monday).
   - **No:** show a friendly **"This app is invite-only"** screen with the email they signed in with, a line such as "Ask John to add this email," and a **Sign out** button. **No raw error codes or permission-denied text.** Also handle other failures (offline, email not verified) with a plain-language message.
4. **All app data paths go through `users/{uid}/...`** using the signed-in user's UID, never a constant. Put path-building in one small module so later handoffs and a future shared calendar can build on it.
5. **README section "Adding or removing a person":** exact console clicks to add a document to `allowedEmails` (lowercase email), to remove one, and what happens to a removed person's old data (it stays under their UID until deleted by John in the console). Remind John that emails must be typed in lowercase.
6. **Prove it works:**
   - Use a second Google account if John has one: before adding it, confirm the invite-only screen shows and no data is readable. Then add it to `allowedEmails`, confirm it signs in and sees an empty private app. Then remove it and confirm it is blocked again.
   - If no second account is available, use the **Rules Playground** in the console to show that (a) another user's UID path is denied, and (b) an email not in `allowedEmails` is denied.
   - Confirm John's own account still works the whole time.

## Step 6: App shell
- A **bottom tab bar** with four tabs: **Calendar, Daily Schedule, Shopping List, To-Do** (use a clear label and icon each). Each tab shows a simple placeholder ("Calendar coming soon").
- A **gear icon** opening a placeholder Settings screen (contains the Sign out button).
- Mobile-first layout, respects the iPhone safe areas (notch and home indicator), and looks fine in a wide laptop browser.
- Follows system light and dark mode.
- Enable **Firestore offline persistence** (IndexedDB) in the Firebase initialization now, so it is in place for the next handoff. Do not read or write any real data yet other than a tiny test if needed for the rules check (remove any test data afterward).

## Step 7: Deploy and the iPhone test (the key milestone)
1. `npm run build` and `firebase deploy --only hosting`. Give John the live URL.
2. Walk John through on his iPhone: open the URL in **Safari**, Share, **Add to Home Screen**, then open it from the Home Screen icon.
3. Verify and report each of these with a clear pass or fail:
   - [ ] App opens full screen (no Safari address bar).
   - [ ] **Google sign-in completes inside the Home Screen app** and lands on the shell.
   - [ ] Closing the app fully and reopening it keeps John signed in.
   - [ ] Airplane mode: the app still opens (offline shell works).
   - [ ] All four tabs and Settings are reachable; the bottom bar does not collide with the home indicator.
   - [ ] Same sign-in works on the Windows laptop browser.
   - [ ] An account **not on the allow-list** sees the friendly invite-only screen and cannot read any data.
   - [ ] An account **added to the allow-list** signs in and gets its own empty private app (and cannot see John's data).
   - [ ] **Removing** that account from the allow-list blocks it again.
   - [ ] John's own `settings` document exists under his UID with the default values.
4. **If sign-in fails in the Home Screen app,** STOP. Do not guess at big changes. Report exactly what happens (error text, screenshots from John) and propose options to John. John will bring it to Craig 2.

## Do NOT do in this handoff
- No events, tasks, calendar views, lists, or recurrence logic.
- No notifications, push, Cloudflare Worker, or widget code.
- No analytics, tracking, or extra third-party services.
- Do not upgrade the Firebase project to Blaze or add a payment method.
- Do not make architecture changes (framework, hosting, database) without asking John.

## Deliverables
- Working repo pushed to private GitHub, deployed at a live Firebase Hosting URL.
- `firestore.rules`, `firebase.json`, `.env.example`, `.gitignore`.
- `README.md` with: how to run locally, how to deploy, where config lives, and how to add or remove an invited person (step 5.5).
- **`CHANGELOG.md` created** with the first entry (date, title "Handoff 001: project setup and Firebase", bullets of what changed, anything left undone or deviating from this handoff).
- A short report to John: the live URL, the pass or fail checklist from step 7, and any questions or deviations.

## Definition of done
All step 7 checks pass (or a failure is clearly reported and John has chosen the next step), the changelog is updated, John can sign in on both his iPhone Home Screen app and his laptop, and the invite-only behavior (blocked, added, removed) has been demonstrated.
