# CLAUDE.md — Ramat Zvi Community Calendar

This file orients Claude Code inside this repo. It captures the full context of the
sessions that built and migrated this app, so work can continue without
re-discovering any of it.

## What this project is

A single-page, static community events calendar for the community of **Ramat Zvi**,
in Hebrew (RTL). It is hosted for free on **GitHub Pages** and stores its event
data in a **Supabase** project (hosted Postgres), using **Google Sign-In (via
Supabase Auth)** to gate who can add/edit/delete events. There is no backend
server of our own — the browser talks straight to Supabase's REST + Auth APIs.

- **Live site:** https://ramatzvi.github.io/calendar/
- **GitHub repo:** `ramatzvi/calendar` (GitHub Pages served from the `main` branch root)
- **Owner / contact:** ofir.landshaft@gmail.com
- **Supabase project:** ref `slsfqlgtthqgkiysywdz`, region `eu-central-1` (Frankfurt)
- **Google Cloud project:** `ramat-zvi-calendar` (only used now as the Google OAuth
  client that Supabase's Google provider points at)

> **History:** the app originally used a Google Sheet as its datastore and Google
> Identity Services (GIS token client) for auth. It was migrated to Supabase to
> get real auth sessions, simpler writes, and — crucially — to drop the sensitive
> `.../auth/spreadsheets` OAuth scope, which had the app stuck in Google's
> unverified-app / verification limbo. `docs/google-oauth-verification.md` is kept
> only as a record of that dead end.

## Why it's built this way

The ask was for a *real* shared calendar (not a per-browser demo) that any
community member can view without signing in, but where only approved people can
edit. The design:

- **Viewing** requires no sign-in — events are read from Supabase with the public
  **publishable key**. A Row Level Security policy (`public read`, `using (true)`)
  allows anonymous `SELECT` on the `events` table and nothing else.
- **Editing** requires signing in with Google (handled by Supabase Auth). The UI is
  gated client-side by `isEditor()` (just "is there a session"), but the *real*
  enforcement is server-side: RLS `INSERT/UPDATE/DELETE` policies on `events` call
  `public.is_editor()`, which is true only if the signed-in email is a row in the
  `editors` table. A non-editor's write fails with Postgres error `42501`.
- **Who can edit** is one list Ofir manages directly — see "Managing editors".

## Tech stack

- Plain HTML/CSS/JS, single file (`index.html`, ~770KB — most of that is three
  base64-embedded background photos). No build step, no framework, no npm.
- Tailwind via CDN (`cdn.tailwindcss.com`) — throws a harmless "should not be used
  in production" console warning; pre-existing, not something to "fix."
- **`@supabase/supabase-js` v2** via CDN
  (`cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.js`), exposes
  `window.supabase`. The app makes one client: `var sb = supabase.createClient(...)`.
- **Supabase Auth** with the **Google** provider, `signInWithOAuth` redirect flow
  (not a popup). Supabase owns the session + token refresh; `onAuthStateChange`
  keeps `authState` in sync.
- **Supabase REST (PostgREST)** as the datastore — `sb.from('events').select/insert/update/delete`.

## Key files

| File | Purpose |
|---|---|
| `index.html` | The entire app: markup, styles, and all JS. |
| `privacy.html` | Hebrew privacy policy — linked from Google's OAuth consent screen. |
| `terms.html` | Hebrew Terms of Service — linked from Google's OAuth consent screen. |
| `google2d2eb4e31d5b5e0c.html` | Google Search Console domain-ownership file. Content is exactly `google-site-verification: google2d2eb4e31d5b5e0c.html`. Harmless to keep; only mattered for the old verification attempt. |
| `.github/workflows/supabase-keepalive.yml` | Daily cron that pings the Supabase REST API so the free-tier project never auto-pauses (see gotcha 3). |

See `docs/architecture.md` for how the app code is organized.

## Supabase schema

Created via the SQL editor (kept here for reference / rebuild):

```sql
create table events (
  id          uuid primary key default gen_random_uuid(),
  title       text not null,
  date        date not null,
  start_time  time not null,     -- "end" is a reserved word, so *_time for both
  end_time    time not null,
  location    text not null,
  description text default '',
  updated_at  timestamptz not null default now(),
  created_by  uuid references auth.users(id) default auth.uid()
);
alter table events enable row level security;

create table editors ( email text primary key );
alter table editors enable row level security;   -- no policy => only dashboard/service role

create function public.is_editor() returns boolean
  language sql security definer stable
  set search_path = ''
as $$ select exists (select 1 from public.editors where email = auth.jwt() ->> 'email') $$;

create policy "public read"    on events for select using (true);
create policy "editors insert" on events for insert to authenticated with check (public.is_editor());
create policy "editors update" on events for update to authenticated using (public.is_editor()) with check (public.is_editor());
create policy "editors delete" on events for delete to authenticated using (public.is_editor());
```

The app maps DB columns `start_time`/`end_time` back to `start`/`end` via a
PostgREST select alias (`start:start_time,end:end_time`) and trims the `HH:MM:SS`
that Postgres `time` returns down to `HH:MM` in `rowToEvent()`.

## Config constants (in `index.html`)

Client-exposed by design (no backend to hide them behind), safe to commit:

```js
var SUPABASE_URL = 'https://slsfqlgtthqgkiysywdz.supabase.co';
var SUPABASE_KEY = 'sb_publishable_zcb1SwmWcKppK9XhZBngUA_VxEkifU-';  // publishable (anon) key
```

The publishable key only grants what RLS allows (public read + editor-gated
writes). The **secret / `service_role`** key must never appear in this repo.

## Managing editors (told to the user, keep this accurate)

**One list:** the `editors` table in the Supabase project. Add a row with someone's
Google email → they can add/edit/delete events. Remove the row → they can't.
Manage it in the Supabase dashboard → **Table editor → editors**, or via SQL:

```sql
insert into editors (email) values ('someone@example.com');   -- grant
delete from editors where email = 'someone@example.com';      -- revoke
```

Signing in still works for *anyone* with a Google account (they just get a
read-only calendar unless their email is in `editors`). There is no test-user cap
and no "unverified app" warning anymore, because the app only requests the
non-sensitive `email` / `profile` / `openid` scopes.

## Google OAuth setup (for reference)

- Supabase dashboard → **Authentication → Providers → Google**: enabled, holds the
  Client ID + Secret of the `ramat-zvi-calendar` Cloud project's Web OAuth client.
- That OAuth client has `https://slsfqlgtthqgkiysywdz.supabase.co/auth/v1/callback`
  in its Authorized redirect URIs.
- Supabase → **Authentication → URL Configuration**: Site URL
  `https://ramatzvi.github.io/calendar/`; Redirect URLs allow-list includes
  `https://ramatzvi.github.io/calendar/**` and `http://localhost:8777/**` (local test).
- Google consent screen scopes: `.../auth/userinfo.email`, `.../auth/userinfo.profile`,
  `openid` — all non-sensitive. The old `.../auth/spreadsheets` scope has been
  removed; if it ever reappears the app is back in verification limbo.

## Deployment

GitHub Pages serves directly from the `main` branch root of `ramatzvi/calendar` —
no build/CI step. Push `index.html` (or the other root HTML files) to `main` and
GitHub's `pages-build-deployment` Action redeploys, usually under a minute.

`.github/workflows/supabase-keepalive.yml` also needs to reach `main` (or wherever
Actions run) to do its job.

When testing on the live URL, GitHub Pages / browser caching can mask a
just-deployed fix — append `?nocache=123` when re-checking.

## Known gotchas (don't re-break these)

1. **RLS must stay enabled on `events` and `editors`.** The publishable key is
   public; if RLS is off or a policy is dropped, anyone can read/write everything.
   Verify with `select relname, relrowsecurity from pg_class where relname in ('events','editors');`.
2. **Never put the `service_role` / secret Supabase key in the repo or the client.**
   Only the publishable key.
3. **Free-tier Supabase projects auto-pause after ~7 days of no DB activity**, and a
   paused project has to be un-paused by hand. `supabase-keepalive.yml` pings it
   daily. GitHub also disables scheduled workflows after 60 days of no repo commits
   — if the repo goes quiet for months, push any commit or add a second pinger
   (e.g. cron-job.org) on the same URL.
4. **Column names are `start_time` / `end_time`**, not `start` / `end` — `end` is a
   Postgres reserved word. The alias trick in `loadEvents()` keeps the rest of the
   app on `start` / `end`.
5. **`public.is_editor()` is `security definer` with `set search_path = ''`** so it
   can read the `editors` table past that table's RLS while staying safe. If you
   recreate it, keep both of those.
6. `isEditor()` in `index.html` is intentionally an *optimistic* client-side check
   (just `authState.signedIn`) — don't "fix" it into something stricter; the real
   check is the RLS write policy.
