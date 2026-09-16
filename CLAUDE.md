# CLAUDE.md

Project instructions for Claude Code. Read `SPEC.md` in this directory before doing
anything — it is the source of truth for what this app is and why each decision was
made. `SETUP.md` is the console work the owner does before Phase 1. This file is
about *how to work here*.

Timezone for all date logic: **America/Chicago**. Scheduler: GitHub Actions, not
Vercel cron. App login: Google sign-in sets a session cookie; only the owner's
Google account ID is accepted.

---

## What this is

A single-user PWA that helps the owner (who has ADHD) remember work and personal
commitments. Captures by voice, text, and photo; reads Gmail and Calendar; chats with
tool use; sends a twice-daily digest. Runs on free infrastructure. One user, one
Google account, one phone — an iPhone.

## Stack

- Next.js on Vercel (Hobby tier, free)
- Supabase — Postgres + Storage (free tier)
- Gemini API for all AI (audio, vision, classification, chat, digest)
- Google OAuth for Gmail read + Calendar
- Apple Shortcuts for Reminders sync

---

## Working agreement

**The owner is new to Claude Code.** Explain what a file does before writing it, in
plain language. Don't assume familiarity with framework conventions.

**Plan before building.** Use plan mode for anything touching more than one file.
Show the plan, let it be argued with, then build.

**Small commits.** After every working step, commit. Git is the undo button here.
Never make a large multi-file change without a commit behind it.

**One phase per session.** "Build Phase 1 from SPEC.md" is a session. "Build the app"
is not. See SPEC.md §11 for the phases.

**Ask when the spec is ambiguous.** Don't invent behavior and move on. A wrong
assumption buried in working code is more expensive than a question.

---

## Hard rules

**Secrets**
- All keys live in `.env.local`. `.env*` is gitignored.
- Never print a key value into the terminal, a log, a comment, or a commit.
- Never hardcode a key as a fallback default.

**Google OAuth**
- The consent screen must be in **Production** publishing status, not Testing.
  Testing-mode refresh tokens expire after 7 days and the app silently dies.
- Handle `invalid_grant` by writing to the `health` table so it surfaces in the
  digest. Never swallow it.

**Gemini**
- The GCP project must have Cloud Billing enabled — the paid API tier. On the free
  tier, prompts and responses are used to improve Google's products, and this app
  handles client emails, addresses, and jobsite photos.
- A consumer Google AI Pro subscription does **not** provide API access. Don't
  suggest otherwise.

**Photos**
- Always compress client-side before upload: ~1600px long edge, JPEG quality 80.
  Never upload an original. Supabase free storage is 1GB.

**Nothing blocks a capture**
- A capture saves immediately. Clarifying questions are non-blocking and sit in the
  thread. If the owner ignores a question, nothing is lost.
- Optimistic UI throughout — the message appears instantly, processing runs behind it.

**Nothing fails silently**
- Every scheduled job, Shortcut, and external call writes to the `health` table.
  Staleness surfaces at the top of the digest. See SPEC.md §8.

**No PWA push notifications**
- Notifications go through Apple Reminders via Shortcuts. This was decided
  deliberately; iOS PWA push is unreliable and fails invisibly.

**Precision over recall**
- When classifying email, err toward missing a task rather than surfacing a
  non-task. A noisy digest kills the app; a missed item does not.

---

## UI rules

One screen. No tabs, no nav bar, no hamburger menu. If a tab seems necessary,
the structure is wrong — raise it rather than adding one.

- Pinned "Now" strip (top 3, collapses on scroll, always expanded on cold open)
- Chat thread in the middle — captures, replies, and digests all live here
- Fixed input bar at the bottom: text, hold-to-talk, camera

Mobile-first, iPhone Safari specifically. Test `MediaRecorder` behavior there before
relying on it; it has historically been quirky. Fall back to
`<input type="file" accept="audio/*" capture>` if needed.

---

## Code conventions

- TypeScript throughout.
- Server-side Supabase access only, using the service role key. No client-side
  database access — there is one user and no row-level security to lean on.
- All Gemini calls go through one wrapper module. Do not scatter API calls through
  the codebase; the prompts will need constant tuning and must be findable.
- Prompts live in their own files under `prompts/`, not inline in handlers.
- Structured output from Gemini gets validated (zod or equivalent) before it touches
  the database. Never trust model JSON directly.
- Wrap every external call in try/catch and write failures to `health`.

---

## What not to do

- Don't add a second AI provider. Gemini only — this was decided after ruling out
  Claude API (no audio input) and a separate transcription service.
- Don't build ambient/background audio capture. iOS does not permit it from a web
  app. Capture is explicitly tap-to-record.
- Don't use the unofficial iCloud CalDAV route for Reminders.
- Don't stuff full conversation history into chat requests. Memory is retrieval-based
  — see SPEC.md §6.
- Don't add features ahead of the current phase, even small ones.

---

## Useful commands

```
npm run dev          # local dev server
npm run build        # production build — run before pushing
npx supabase ...     # database CLI
```
