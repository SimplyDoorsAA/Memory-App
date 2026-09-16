# Personal Memory & Assistant App — Build Spec

Single-user PWA for iPhone. Capture thoughts by voice, text, or photo; chat with an
assistant that knows your Gmail, Calendar, and task list; get a twice-daily digest.
Runs on free infrastructure. One user: the owner.

---

## 1. Problem being solved

The owner has ADHD, task-switches frequently, and loses commitments made in
conversation, email, and passing thoughts. Commercial AI memory wearables (Fieldy,
Plaud, Bee, Limitless) solve capture but not review — the transcript pile goes unread.

**This app inverts that: review is the product, capture is the feeder.**

Every design decision below follows from two rules:
1. **Nothing may block a capture.** If the app makes you wait, confirm, or answer a
   question before a thought is saved, you will lose the thought.
2. **Nothing may fail silently.** This app's job is noticing what you'd miss. It
   cannot be the thing that breaks without telling you.

---

## 2. Locked decisions

| Area | Decision |
|---|---|
| AI provider | **Gemini only** (paid API tier). Handles audio, vision, classification, chat, digest |
| Hosting | Vercel (Hobby, free) |
| Database + storage | Supabase (free tier — Postgres + 1GB storage) |
| Auth | Google OAuth, **one account** (work + personal in one Gmail) |
| App login | The same Google sign-in sets a long-lived signed session cookie. Only the owner's Google account ID is accepted; any other account is rejected |
| Client | PWA installed to iPhone home screen |
| Notifications | **Apple Reminders**, via Shortcuts. No PWA push |
| Digest delivery | Appears as a message in the app's chat thread; a recurring Reminder with a URL opens it. Email in parallel during validation |
| Digest schedule | Morning ~6:30am, evening ~6:00pm, **America/Chicago** |
| Scheduler | **GitHub Actions** `schedule:` workflow in this repo, hitting the Vercel endpoint with `CRON_SECRET`. Not Vercel cron (§3.3) |
| Digest email | Resend free tier, no custom domain. Account created with the digest recipient Gmail; sender is `onboarding@resend.dev` |
| Digest format | Two-tier: top 3 first, everything else below |
| Capture modes | Voice (hold-to-talk), text, photo |
| Task sync | **Two-way** with Apple Reminders. Matching key is the item UUID written into the Reminder's Notes field (§7) |
| Chat memory | Retrieval-based, not full-context (see §6) |
| Chat abilities | Tool use / function calling — it can actually create, complete, and reschedule |

**Why Gemini only:** it takes audio natively, so voice capture and conversation go
straight to the model with no separate transcription service. One vendor, one OAuth,
one billing account, one set of data terms. Claude API was considered and dropped —
it cannot accept audio input.

---

## 3. Critical gotchas — read before writing code

Each of these silently breaks the app if ignored.

### 3.1 Google OAuth must be in **Production**, not Testing
External OAuth apps in **Testing** status with sensitive scopes have refresh tokens
that **expire after 7 days**. The cron would die every week with `invalid_grant`.

- Google Cloud Console → APIs & Services → OAuth consent screen → Audience →
  **Publish App**.
- This is *separate from verification*. Unverified-but-Production shows a "Google
  hasn't verified this app" screen once; click Advanced → Go to (app). Tokens then
  persist.
- Full verification (CASA assessment for restricted Gmail scopes) takes weeks and is
  **not needed** for one self-authorized user. Do not start it.
- Handle `invalid_grant` by surfacing "reconnect Google" in the digest, never by
  failing quietly.
- **Verify empirically in Phase 1 step 1:** `gmail.readonly` is a *restricted* scope,
  not merely sensitive. Confirm Google lets an unverified External app in Production
  request it for the owner's own account. If it refuses, the fallback is a Google
  Workspace account (Internal app type), which changes the one-consumer-Gmail decision.

### 3.2 Gemini paid tier requires Cloud Billing — the Google AI Pro subscription does NOT cover it
A consumer Google One / Google AI Pro subscription gives Gemini in Gmail, Docs, and
AI Studio. It does **not** upgrade the Gemini API tier.

- Enable **Cloud Billing** on the GCP project.
- This matters for data handling: on the free API tier, prompts and responses are
  used to improve Google's products. On pay-as-you-go, they are not. Client emails,
  names, addresses, and jobsite photos flow through this app — use the paid tier.
- Expected cost at personal volume with Flash: a few dollars a month.

### 3.3 Vercel Hobby cron — not used
Hobby tier allows two cron jobs, each at most once per day, and fires them anywhere
within the scheduled hour. A digest landing at 7:20 when the Reminder fires at 6:30
is a broken experience.

**Decision: GitHub Actions** `schedule:` workflow in this repo, hitting
`/api/digest` over HTTPS with `CRON_SECRET` in a header. Schedules are written in UTC;
America/Chicago needs **two entries per digest** (CST and CDT offsets) with the
endpoint rejecting the one that's off by an hour based on local time. GitHub Actions
can also drift a few minutes at busy times; that's acceptable, an hour is not.

### 3.4 Apple Reminders has no server-writable API
No Apple API, no OAuth, no REST endpoint. Sync happens through two Apple Shortcuts on
the owner's phone (§7). Do **not** use the unofficial iCloud CalDAV route — it's
undocumented and Apple can break it without notice.

Shortcuts also does not expose a stable Reminder identifier. Matching is done by
writing the item UUID into the Reminder's Notes field (§7).

### 3.5 No background audio on iOS web
A PWA can use mic and camera while open and in the foreground. It **cannot** listen
in the background or from the lock screen. Capture is explicitly tap-to-record.
Do not design around ambient capture.

### 3.6 Photo storage math
Supabase free tier = 1GB. An iPhone photo is 2–4MB, so ~300 photos before it's full.
**Compress client-side before upload**: resize to ~1600px long edge, JPEG quality 80
→ 200–400KB. That's thousands of photos, still perfectly readable for OCR and for
looking at later. Do this in-browser with a canvas; never upload the original.

### 3.7 Supabase free tier pauses after ~7 days inactivity
The daily cron prevents this in normal use. After a long vacation it may need a
manual wake.

### 3.8 Shortcuts automations fail silently
iOS kills scheduled automations for many reasons — phone off, low power mode, an
unseen permission prompt, an OS update. There is no error. See §8 for the required
staleness detection.

---

## 4. Architecture

```
         ┌──────────────── iPhone ────────────────┐
         │  PWA (single screen)                    │
         │   · pinned "Now" strip                  │
         │   · chat thread                         │
         │   · capture bar (text / mic / camera)   │
         │                                          │
         │  Shortcuts                               │
         │   · push:  GET  /api/reminders/pending   │
         │            (UUID goes into Reminder Notes)│
         │   · sync:  POST /api/reminders/completed │
         │   · recurring Reminder w/ app URL        │
         └──────────────────┬───────────────────────┘
                            │
                     Vercel serverless
                            │
   ┌────────────────────────┼────────────────────────┐
   │                        │                        │
Gemini API              Supabase              Google APIs
· audio → text       · items, notes         · Gmail (read)
· image → text       · conversations        · Calendar (r/w)
· classification     · profile
· chat + tool use    · Storage (photos)
· digest writing
```

---

## 5. Data model

```sql
-- Everything captured or ingested
create table items (
  id            uuid primary key default gen_random_uuid(),
  created_at    timestamptz not null default now(),
  kind          text not null default 'task',   -- 'task' | 'note'
  source        text not null,                  -- 'voice' | 'text' | 'photo'
                                                -- | 'gmail' | 'calendar' | 'chat'
  raw_content   text,                           -- transcript / typed / OCR / snippet
  title         text not null,
  detail        text,
  job           text,                           -- project or client, nullable
  due_at        timestamptz,
  status        text not null default 'open',   -- 'open' | 'done' | 'dismissed'
  needs_detail  boolean not null default false, -- low-confidence date or job
  confidence    real,                           -- model's own 0–1 on its extraction
  source_ref    text,                           -- gmail message id / calendar event id
  pushed_at     timestamptz,                    -- when sent to Reminders
  completed_via text,                           -- 'app' | 'reminders' | 'digest'
  feedback      text                            -- 'not_a_task' | 'wrong_date' | null
);

create table attachments (
  id           uuid primary key default gen_random_uuid(),
  item_id      uuid references items(id) on delete cascade,
  storage_path text not null,
  thumb_path   text,
  created_at   timestamptz not null default now()
);

-- Chat thread. Digests are stored here too, as assistant messages.
create table messages (
  id         uuid primary key default gen_random_uuid(),
  created_at timestamptz not null default now(),
  role       text not null,              -- 'user' | 'assistant'
  content    text not null,
  message_kind text default 'chat',      -- 'chat' | 'digest_morning' | 'digest_evening'
                                         -- | 'review_weekly' (Phase 3)
  item_ids   uuid[]
);

-- Rolling durable facts about the owner: projects, people, phrasing preferences.
-- Small. Injected into every chat turn.
create table profile (
  id         int primary key default 1,
  content    text not null,
  updated_at timestamptz not null default now()
);

create table google_auth (
  id            int primary key default 1,
  google_sub    text not null,                  -- owner's Google account ID; login rejects any other
  refresh_token text not null,
  updated_at    timestamptz not null default now()
);

-- Heartbeats for silent-failure detection (§8)
create table health (
  key       text primary key,   -- 'cron_digest' | 'shortcut_push' | 'shortcut_sync'
                                -- | 'gmail_sync' | 'google_auth'
  last_ok   timestamptz,
  last_error text
);
```

Index `items` on `(status, due_at)`, `(kind, created_at)`, `(job)`.

---

## 6. Chat memory — retrieval, not full context

Do **not** stuff conversation history into every request. Cost and latency grow
without bound, and answer quality degrades as today's three real tasks get buried
under months of chatter.

Every chat turn receives:
1. **Live state** — open tasks, today + tomorrow's calendar, and items with
   `source='gmail'` created in the last 48h (the "recently flagged emails").
   Small, fresh, always included.
2. **Profile** — the `profile` table. Durable facts: projects, people, how the owner
   phrases things. Updated by the model when it learns something lasting. Cheap.
3. **Recent thread** — last ~20 messages for conversational continuity.
4. **Retrieved history on demand** — when the owner references something older
   ("what did I say about the Hoelscher order last month"), a search tool pulls the
   matching items and messages.

This feels like full memory while staying fast and cheap indefinitely.

---

## 7. Apple Shortcuts integration (two-way)

All endpoints authenticate with a static bearer token (`SHORTCUTS_BEARER_TOKEN`).
Single user, so this is sufficient.

Shortcuts does not expose a stable identifier for a Reminder it creates, so the app
never learns an Apple ID. Instead the **item UUID is written into the Reminder's Notes
field** and read back from there. No `registered` endpoint is needed.

### Shortcut A — push tasks into Reminders
`GET /api/reminders/pending`
- Returns `[{id, title, detail, due_at}]` where `status='open' AND pushed_at IS NULL`.
- Sets `pushed_at` on returned rows so they aren't duplicated next run.
- Shortcut steps: Get Contents of URL → Get Dictionary from Input → Repeat with Each
  → Add New Reminder (title = `title`, alert at `due_at`, **Notes = `id`** followed by
  `detail` on the next line).

### Shortcut B — sync completions back
`POST /api/reminders/completed` with `{item_ids: [...]}`
- Finds Reminders where Is Completed is true, extracts the first line of Notes from
  each, POSTs the resulting list.
- App marks matching items `status='done'`, `completed_via='reminders'`. Unknown IDs
  are ignored. Already-done IDs are idempotent.
- **Without this, the two lists drift apart within a week and the app starts nagging
  about finished work.** This is not optional.

### Recurring digest Reminder
Two repeating Reminders (6:30am, 6:00pm) titled "Morning digest" / "Evening digest"
with the app URL attached. Apple delivers the alert; tapping opens the PWA to the
digest message. This is why PWA push isn't needed.

Every Shortcut run updates `health` (§8).

---

## 8. Health checks — surfaced in the digest

The digest is the one surface the owner looks at daily, so all health signals belong
there. If any of these is stale, the digest leads with it before anything else:

| Check | Stale after | Message |
|---|---|---|
| `shortcut_sync` | 24h | "Reminders sync hasn't run since [date]" |
| `shortcut_push` | 24h | "Tasks haven't reached Reminders since [date]" |
| `gmail_sync` | 12h | "Haven't been able to read email since [date]" |
| `google_auth` | on `invalid_grant` | "Reconnect Google" + link |
| `cron_digest` | on app open | see below — the email cannot catch this |

**Missed digest.** The email is sent by the same cron that would have died, so it is
not a fallback for the cron itself. Detection happens **on app open**: if no
`digest_morning` message exists for today after 6:30am local, or no `digest_evening`
after 6:00pm local, the app shows "No [morning/evening] digest was generated" above
the thread. The recurring Reminder opens the app at exactly those times, so this is
seen within seconds of the failure. In Phase 1 the same check runs on the crude page.

---

## 9. UI — one screen

No tabs. No nav bar. No hamburger menu. Every navigation choice is a decision point,
and decision points are where attention leaks. If a tab feels necessary, the
structure is wrong.

**Top — "Now" strip.** Top 3 items, tap to complete. Collapses on scroll to a thin
bar reading e.g. "3 open · 1 overdue", expands on tap. **On cold open, always starts
expanded and scrolled to top** — never restored to yesterday's scroll position.

**Middle — the thread.** A scrolling conversation. Captures appear as user messages;
assistant replies and both daily digests appear as assistant messages. Photos render
as thumbnails, full image on tap. Older days collapse visually.

**Bottom — fixed input bar.** Text field, hold-to-talk mic, camera. Always within
thumb reach.

The full task list is behind a tap on the "Now" header. Settings are buried.

### Two speeds on one input
- **Quick capture** — fire and forget. Confirms in under a second, processes in the
  background, expects no reply. This is the default for a bare statement.
- **Conversation** — waits for an answer. This is what a question gets.

Optimistic UI throughout: the message appears instantly, processing happens behind it.

---

## 10. Capture behavior

### Classification
Each capture goes to Gemini with a structured-output prompt returning:
`{kind, title, detail, job, due_at, confidence}` where `kind` is `task` or `note`.

### Follow-up questions
The model can nearly always *guess* a date and a job, so "ask when it can't" would
mean never asking. Use a **confidence threshold** instead — ask when its own
confidence is near a coin flip. Expect to tune this in the first weeks.

**The question is never a gate.** The item saves immediately with
`needs_detail=true`; the question sits in the thread. Ignore it and nothing is lost —
the item is already on the list, just fuzzy. Answer it and it sharpens.

Unanswered `needs_detail` items batch into the evening digest as "three things I
couldn't place" — cleaned up once, rather than three interruptions during the day.

### Notes vs tasks
Notes stay off the task list, but searchable-only notes become a graveyard — you
can't search for what you've forgotten you saved. Two required countermeasures:
- **Contextual surfacing.** When a task for a given `job` is shown, notes tagged to
  that job come with it. The note finds you.
- **Resurfacing pass.** The weekly review includes a couple of untouched older notes.

Plus: a **one-tap "make this a task"** on any note. The classifier will misfire, and
if fixing that means retyping, the owner stops trusting the split.

### Photos
Compress client-side (§3.6) → Gemini vision for OCR and extraction → store both
compressed image and thumbnail in Supabase Storage, attached to the item.

---

## 11. Build phases

### Phase 1 — the digest loop
Goal: twice-daily digest landing where the owner will see it, assembled from Gmail,
Calendar, and a crude capture.

0. Console setup — see `SETUP.md`. GCP project with Cloud Billing and a budget
   alert, Supabase project, Vercel project, Resend account. None of this exists yet.
1. Google OAuth (Production status, §3.1). Scopes: `openid`, `gmail.readonly`,
   `calendar.readonly`. The same flow logs the owner into the app: on callback,
   compare the Google account ID to `google_auth.google_sub` and set a long-lived
   signed session cookie. The crude capture page and the digest link both sit behind
   this cookie.
2. `/api/sync` — pull last 24–48h of Gmail. Send subject + snippet (not full body) to
   Gemini Flash: *does this contain a date, deadline, or a commitment the owner owes
   someone?* Structured JSON out. Write to `items`.
3. Calendar read for today + tomorrow.
4. **A crude capture box.** Plain textarea, no AI, writes straight to `items`. Ugly
   is fine. Without it the digest contains only email and calendar, which tests the
   wrong thing.
5. `/api/digest?kind=morning|evening` — Gemini writes the two-tier digest (§12),
   stored as a message and emailed.
6. GitHub Actions schedule for both (§3.3).
7. A crude "not a task" link in the digest.
8. Missed-digest check on the crude page (§8).

**Exit criterion: the owner opens the digest five days running without being
reminded to.** If not, fix the digest before building anything else. Do not proceed
on a foundation that isn't landing.

### Phase 2 — the app
1. PWA shell, installable, single screen (§9).
2. Full capture: voice via Gemini audio, text, photo with compression.
3. Chat with **tool use** — `create_item`, `complete_item`, `reschedule_item`,
   `search_history`, `update_profile`. Without tool use this is a worse version of
   the Gemini app; the tools are the point.
4. Retrieval-based memory (§6).
5. Digest moves into the thread; email continues in parallel.

### Phase 3 — the loop closes
1. Shortcuts push + sync-back (§7), health checks (§8).
2. Calendar write-back ("remind me Thursday" → real event). Add `calendar.events`.
3. Weekly review: carryover patterns, resurfaced notes, what never gets done.
4. Retire the parallel email once Reminders delivery has proven reliable for a month.

---

## 12. Digest prompt — starting point

```
You are writing a twice-daily digest for one person who has ADHD, manages door
installation projects at SimplyDoors, and loses track of commitments.

{if any health check is stale: lead with that warning before anything else}

Write exactly this structure:

**TOP 3**
The three things that matter most. One line each, verb first. Anything overdue, or
where someone is actively waiting, goes here.

**EVERYTHING ELSE**
Remaining open items, grouped however is clearest for today's set — by job, by
person, or by day. One line each.

{evening only, if any exist}
**COULDN'T PLACE**
Items captured today that need a date or a job.

Rules:
- No preamble, no encouragement, no "you've got this."
- Specific: "Call Robert re: door hardware quote" not "follow up on quote."
- If an item has a date, say the date.
- If nothing is urgent, say so plainly rather than inflating something.
- {evening only} Open by naming what carried over from this morning.

Items: {items_json}
```

Tune this aggressively over the first two weeks. It is the highest-leverage file in
the project.

---

## 13. The failure mode to design against

**Over-triggering.** "Anything with a date, deadline, or commitment" will catch every
newsletter with a sale date and every automated appointment confirmation. Three noisy
digests and the owner stops opening them — at which point the app is dead no matter
how well everything else works.

Defenses, in priority order:
1. **"Not a task" feedback** from day one, even crudely.
2. Feed dismissed items back into the classification prompt as negative examples.
3. **Bias toward precision over recall.** A missed task is recoverable. A noisy
   digest is fatal.
4. Hard-filter obvious automated senders (noreply@, marketing domains) before they
   reach the classifier at all.

---

## 14. Environment variables

```
GOOGLE_CLIENT_ID
GOOGLE_CLIENT_SECRET
GOOGLE_REDIRECT_URI
SESSION_SECRET              # signs the app login cookie
GEMINI_API_KEY              # from a Cloud-Billing-enabled GCP project
SUPABASE_URL
SUPABASE_SERVICE_ROLE_KEY
RESEND_API_KEY              # digest email during validation
DIGEST_RECIPIENT_EMAIL
CRON_SECRET
SHORTCUTS_BEARER_TOKEN
APP_BASE_URL
TZ=America/Chicago          # all "today", "morning", "evening" logic uses this
```

---

## 15. Cost expectation

| Item | Cost |
|---|---|
| Vercel Hobby | $0 |
| Supabase free tier | $0 |
| Resend free tier | $0 |
| GitHub Actions (scheduler) | $0 |
| Gemini API (paid tier, personal volume) | a few dollars/month |

Set a budget alert on the GCP project before first deploy. There is no hard spending
cap by default, and a looping bug on a cron endpoint has nothing to stop it.
