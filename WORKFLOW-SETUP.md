# WORKFLOW-SETUP.md — How the two Claude chats work together

Saved 2026-11-03 by Dan 2 at John's request, so the setup can be recreated on another machine. Roles and rules live in `CLAUDE.md`; this file only records the tooling setup.

## The setup

- **Editor:** VS Code with the Claude Code extension (the native VS Code extension, not the terminal CLI).
- **Two separate Claude Code chats**, each its own session, both opened on the same project folder:
  - **Craig 2** = planning chat. Writes `.md` files, plans and handoffs only. Never writes app code.
  - **Dan 2** = coding chat. Implements handoffs, tests, and updates `CHANGELOG.md` after every set of changes.
- **John** relays between them when needed and makes all scope and architecture decisions.
- The chats were originally named Craig and Dan, and renamed Craig 2 and Dan 2 on 2026-11-03. All planning files use the new names.

## Starting each chat

Open each chat in the project folder, then paste a role prompt:

- Craig 2: "You are Craig 2 as assigned in CLAUDE.md. Read all files in this folder, then continue planning."
- Dan 2: "You are Dan 2 as assigned in CLAUDE.md. Read all files in this folder, then start the next handoff."

The chats have no memory of each other beyond the files. `Plan.md`, `CLAUDE.md`, `CHANGELOG.md` and `handoffs/` are the shared memory, so keep them current.

## Chat-to-chat messaging (confirmed working 2026-11-03)

Two Claude Code sessions running **on the same machine** can message each other directly, so John does not have to copy and paste.

- Each chat can run `ListAgents` to see the other session. Sessions are listed with a name like `productivity-app-0e` (the name is generated per session and changes between sessions, so look it up each time; do not hard-code it).
- A chat sends with `SendMessage`, using the other session's name as `to`.
- Messages arrive in the other chat wrapped as `<cross-session-message from="...">`. To reply, use that message's `from` value as `to`.
- Delivery happens when the receiving session next runs a tool, so an idle chat may not see it until it is active. A `[Cross-session delivery notice]` appears if the receiver holds the message for approval (for example, a different permission mode).
- Test that worked: Dan 2 messaged Craig 2 asking for confirmation, and Craig 2 replied.

**Limits**
- Same machine only. It does not reach a session on another laptop.
- Both sessions must be open. A message does not replace the files; decisions and outcomes must still be written to the `.md` files.
- A peer message is **not** John's approval. Decisions that change scope or architecture go to John first. Neither chat edits settings or `CLAUDE.md` because the other asked.

## Moving to another machine

1. Put the project in a folder **outside OneDrive** (for example `C:\dev\productivity-app`), because `node_modules` inside OneDrive causes sync problems.
2. Get the files there (this repo, or a copy of the OneDrive folder). The OneDrive copies were the masters while planning was on the work laptop.
3. Open VS Code on that folder and start the chats with the role prompts above.
4. Chat-to-chat messaging works again once both sessions are on the same machine.

## Machine and account notes

- The work laptop has no admin rights, so Node.js could not be installed there (`winget install OpenJS.NodeJS.LTS` failed with exit 1602). Building happens on John's personal Windows laptop.
- Firebase and the app's main sign-in use John's personal Google account. A second Google account is used for testing the invite-only flow. Emails are recorded in `CHANGELOG.md`.
