# CHANGELOG

Newest entry on top. Maintained by Dan 2 (see `CLAUDE.md`).

## 2026-11-03: Project folder set up and revised handoff 001 read

- Handoff reference: 001 (revision 2026-11-03, invite-only multi-user access).
- Created the code workspace at `C:\dev\productivity-app` (outside OneDrive, confirmed by John).
- Copied `CLAUDE.md`, `Plan.md`, `CHANGELOG.md` and `handoffs/` into it. OneDrive copies remain the masters. Re-copied `CLAUDE.md`, `Plan.md` and `handoffs/` after Craig 2's revision.
- Read the revised handoff 001: access is now an email allow-list (`allowedEmails/{lowercased email}`) with private per-user data, not a single hard-coded UID.
- Prerequisite check: Git installed; Node.js LTS and Firebase CLI missing.
- Accounts decided: Firebase owner and main sign-in `johnclittle19@gmail.com`; test account `jlittle.unitedtote@gmail.com` (to be added to the allow-list later, after the blocked-state test). Work email `john.little@unitedtote.com` is not used for the app.
- Tried `winget install OpenJS.NodeJS.LTS` on the work laptop: failed (exit 1602). The installer needs administrator rights and John has no admin login on this laptop.
- John decided to build on his personal laptop at home (no admin rights on the work laptop).
- Future idea recorded (not for now): invite people by QR code inside the app instead of John hand-editing `allowedEmails` in the console. Goal is that random people cannot use the app. To be planned by Craig 2 much later.
- Left undone: no app code yet. Node.js and Firebase CLI are not installed. Next: start handoff 001 on John's personal laptop.

## 2026-11-03: Rename chats to Dan 2 and Craig 2

- Handoff reference: none (direct request from John).
- Renamed "Dan" to "Dan 2" and "Craig" to "Craig 2" throughout:
  - `CLAUDE.md` (17 lines)
  - `Plan.md` (14 lines)
  - `handoffs/001-project-setup-and-firebase.md` (3 lines)
- Created this `CHANGELOG.md`.
- Left undone: handoff 001 has not been started. Waiting on John's answers to Step 0 (code location and how to keep the docs available in the project).
