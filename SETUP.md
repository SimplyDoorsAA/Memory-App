# Phase 1 console setup

Everything here is done in a browser, by the owner, before any code runs. Work top
to bottom. Each step ends with a value that goes into `.env.local` (never committed)
and later into Vercel's environment variables.

Timezone for everything: **America/Chicago**.

---

## 1. Google Cloud project

1. console.cloud.google.com → New project → name it `memory-app`.
2. **Billing → Link a billing account.** This is required for the paid Gemini API
   tier. A Google One / Google AI Pro subscription does not count.
3. **Billing → Budgets & alerts → Create budget.** $10/month, email alert at 50%,
   90%, 100%. There is no hard cap by default; a looping cron has nothing to stop it.
4. **APIs & Services → Library.** Enable: Gmail API, Google Calendar API,
   Generative Language API.
5. **APIs & Services → OAuth consent screen.**
   - User type: **External**.
   - App name, support email, developer email: yours.
   - Scopes: `openid`, `.../auth/gmail.readonly`, `.../auth/calendar.readonly`.
   - **Audience → Publish App.** Status must read **In production**, not Testing.
     Testing-mode refresh tokens expire after 7 days. Do **not** start verification.
6. **APIs & Services → Credentials → Create credentials → OAuth client ID.**
   - Type: Web application.
   - Authorized redirect URIs: `http://localhost:3000/api/auth/callback` now; add
     the Vercel URL version after step 4 below.
   - Record → `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`,
     `GOOGLE_REDIRECT_URI`.
7. **Credentials → Create credentials → API key.** Restrict it to the Generative
   Language API. Record → `GEMINI_API_KEY`.
   Confirm at aistudio.google.com → your key → the project shows a **paid** tier.

**Empirical check to do at step 5:** after publishing, run the OAuth flow once with
your own account. If Google refuses the `gmail.readonly` scope for an unverified
app rather than showing the "unverified app" warning, stop and reread SPEC.md §3.1.

## 2. Supabase project

1. supabase.com → New project → name `memory-app`, region closest to Chicago
   (US East). Save the database password somewhere safe; it's needed once.
2. **Project Settings → API.** Record → `SUPABASE_URL` (Project URL) and
   `SUPABASE_SERVICE_ROLE_KEY` (service_role, **not** anon).
3. **Storage → New bucket** named `photos`, private.
4. Free tier pauses after ~7 days without traffic. The digest cron keeps it awake.

## 3. Resend

1. resend.com → sign up **with the same Gmail that will receive the digest**. Without
   a verified domain, Resend only delivers to the account's own address.
2. **API Keys → Create.** Record → `RESEND_API_KEY`.
3. `DIGEST_RECIPIENT_EMAIL` = that Gmail. Sender will be `onboarding@resend.dev`.

## 4. Vercel

1. vercel.com → Add New Project → import this GitHub repo. Framework: Next.js.
   Hobby plan.
2. Note the deployment URL → `APP_BASE_URL`. Go back to Google step 6 and add
   `https://<that-url>/api/auth/callback` as a second redirect URI.
3. **Settings → Environment Variables.** Add every variable in SPEC.md §14.
4. Vercel cron is **not** used (SPEC.md §3.3).

## 5. Generate the secrets you own

Run locally, once each, and record the output:

```
openssl rand -hex 32   # CRON_SECRET
openssl rand -hex 32   # SHORTCUTS_BEARER_TOKEN
openssl rand -hex 32   # SESSION_SECRET
```

## 6. GitHub Actions scheduler

Needs two repository secrets: **Settings → Secrets and variables → Actions**:
`CRON_SECRET` (same value as above) and `APP_BASE_URL`. The workflow file is
written in Phase 1 step 6.

## 7. iPhone

Nothing until Phase 3, except: create two repeating Reminders now, "Morning digest"
at 6:30am and "Evening digest" at 6:00pm, each with `APP_BASE_URL` in the URL field.
This starts the five-day open-the-digest habit test the moment Phase 1 ships.

---

## Done when

`.env.local` has every variable from SPEC.md §14 filled in, Google OAuth status
reads "In production", the GCP budget alert exists, and Vercel shows the same
variables. Then start a Phase 1 build session.
