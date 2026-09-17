# Phase 1 console setup

Everything here is done in a browser, by the owner, before any code runs. Work top
to bottom. Each step ends with a value that goes into `.env.local` (never committed)
and later into Vercel's environment variables.

Timezone for everything: **America/Chicago**.

---

## 1. Vercel — deploy the empty app first

The Google consent screen cannot be published until Branding has an application
home page on an authorized domain. That URL comes from Vercel, so this is step one.

1. The repo already contains a bare Next.js app that builds to a one-line page.
2. vercel.com → sign in with GitHub → Add New Project → import `Memory-App`.
   Framework: Next.js. Hobby plan. Deploy with defaults.
3. Note the production URL, something like `memory-app.vercel.app` →
   `APP_BASE_URL` = `https://memory-app.vercel.app`.
4. **Settings → Environment Variables** is where every value from SPEC.md §14 goes
   once you have it. Come back here at the end.
5. Vercel cron is **not** used (SPEC.md §3.3).

## 2. Google Cloud project

1. console.cloud.google.com → New project → name it `memory-app`.
2. **Billing → Link a billing account.** This is required for the paid Gemini API
   tier. A Google One / Google AI Pro subscription does not count.
3. **Billing → Budgets & alerts → Create budget.** $10/month, email alert at 50%,
   90%, 100%. There is no hard cap by default; a looping cron has nothing to stop it.
4. **APIs & Services → Library.** Enable: Gmail API, Google Calendar API,
   Generative Language API.
5. **Google Auth Platform → Branding.** Publish is greyed out until this is done.
   - App name, user support email, developer contact email: yours. No logo
     (uploading one triggers a verification requirement).
   - Application home page: `APP_BASE_URL` from step 1.
   - Privacy policy: `APP_BASE_URL/privacy`. Terms of service: `APP_BASE_URL/terms`.
     Both pages exist in the app. Open them in a browser first; Google rejects
     links that do not load.
   - Authorized domains → Add domain: `memory-app.vercel.app` (your actual
     Vercel host, without `https://`).
   - Save.
6. **Google Auth Platform → Audience.**
   - User type: **External**.
   - **Publish app.** Status must read **In production**, not Testing.
     Testing-mode refresh tokens expire after 7 days. Do **not** start verification.
7. **Google Auth Platform → Data Access.** Add scopes: `openid`,
   `.../auth/gmail.readonly`, `.../auth/calendar.readonly`. Save.
8. **Google Auth Platform → Clients → Create client.**
   - Type: Web application.
   - Authorized redirect URIs, both:
     `https://<your-vercel-host>/api/auth/callback` and
     `http://localhost:3000/api/auth/callback`.
   - Record → `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`.
     `GOOGLE_REDIRECT_URI` is the Vercel one.
9. **APIs & Services → Credentials → Create credentials → API key.** Restrict it to the Generative
   Language API. Record → `GEMINI_API_KEY`.
   Confirm at aistudio.google.com → your key → the project shows a **paid** tier.

**Empirical check to do at step 6:** after publishing, run the OAuth flow once with
your own account. If Google refuses the `gmail.readonly` scope for an unverified
app rather than showing the "unverified app" warning, stop and reread SPEC.md §3.1.

## 3. Supabase project

1. supabase.com → New project → name `memory-app`, region closest to Chicago
   (US East). Save the database password somewhere safe; it's needed once.
2. **Project Settings → API.** Record → `SUPABASE_URL` (Project URL) and
   `SUPABASE_SERVICE_ROLE_KEY` (service_role, **not** anon).
3. **Storage → New bucket** named `photos`, private.
4. Free tier pauses after ~7 days without traffic. The digest cron keeps it awake.

## 4. Resend

1. resend.com → sign up **with the same Gmail that will receive the digest**. Without
   a verified domain, Resend only delivers to the account's own address.
2. **API Keys → Create.** Record → `RESEND_API_KEY`.
3. `DIGEST_RECIPIENT_EMAIL` = that Gmail. Sender will be `onboarding@resend.dev`.

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
