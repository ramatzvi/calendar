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

## Current feature set (status snapshot)

Everything below is built, deployed, and working on the live site as of the
latest session. Full implementation detail for each lives in
`docs/architecture.md`; this is just the at-a-glance list.

- **Views**: month / week / day, plus an all-day band pinned above the hour
  grid in week/day. Clicking a day (or an hour slot) never opens the add
  form — it only *selects* that day (yellow `.is-selected` highlight, same
  pattern as `.is-today`). Switching to week/day follows the current
  selection instead of wherever the view last was.
- **Add**: the only entry point is `#addEventBtn` in the header (hidden
  unless `isEditor()`), defaulting to whatever's selected. Supports a
  one-off event or a **recurring** one (daily/weekly/monthly/yearly) — see
  gotcha 7 and "Recurring events" in `docs/architecture.md`. Warns (with an
  override option) on a same-date/location/overlapping-hours conflict,
  including against all-day events (an all-day event conflicts with
  anything that day at that location).
- **Edit/delete a series member**: a "this / this-and-following / all"
  scope chooser appears first. The bulk paths can also *shift* every
  affected occurrence's date by the same offset if the date field is
  changed, instead of collapsing them onto one date.
- **Access**: a guest (not signed in) sees a static "צפייה בלבד" badge; a
  signed-in non-editor sees the same badge via `refreshCanEdit()`, instead
  of only discovering it after a failed save.
- **Change log**: every insert/update/delete on `events` is mirrored to a
  Google Sheet — see "Change log (audit trail)" below.
- **Locations** (9): בית העם, בית אופיר, חורשת נועם, מגרש, דשא מרכזי,
  מועדון, בית כנסת, השכרת ציוד, and `אחר` (reveals a free-text field).
- **Month-view chips**: intentionally **no truncation** — a long title wraps
  onto as many lines as it needs (no `line-clamp`), matching a reference
  calendar app the user shared. Cells grow taller for long titles,
  especially on narrow/mobile widths — an accepted tradeoff, not a bug.

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
  start_time  time,              -- NULL start/end => an all-day event ("end" is a
  end_time    time,              -- reserved word, hence *_time for both)
  location    text not null,
  description text default '',
  updated_at  timestamptz not null default now(),
  created_by  uuid references auth.users(id) default auth.uid(),
  series_id   uuid           -- shared by every occurrence of a recurring event;
);                            -- NULL for a one-off. No FK, no separate table —
                              -- it's just a grouping key for bulk update/delete.
alter table events enable row level security;
create index events_series_id_idx on events (series_id);

create table editors ( email text primary key );
alter table editors enable row level security;
-- an authenticated user may read ONLY their own editors row; the app uses
-- this to show "view only" vs edit controls up front (refreshCanEdit()).
create policy "read own editor row" on editors for select to authenticated
  using (email = (auth.jwt() ->> 'email'));

create function public.is_editor() returns boolean
  language sql security definer stable
  set search_path = ''
as $$ select exists (select 1 from public.editors where email = auth.jwt() ->> 'email') $$;

create policy "public read"    on events for select using (true);
create policy "editors insert" on events for insert to authenticated with check (public.is_editor());
create policy "editors update" on events for update to authenticated using (public.is_editor()) with check (public.is_editor());
create policy "editors delete" on events for delete to authenticated using (public.is_editor());
```

The `editors` email must match the signed-in user's Google address **exactly**
(lower-case, no whitespace) — that string is what `auth.jwt() ->> 'email'` returns.

The app maps DB columns `start_time`/`end_time` back to `start`/`end` via a
PostgREST select alias (`start:start_time,end:end_time`) and trims the `HH:MM:SS`
that Postgres `time` returns down to `HH:MM` in `rowToEvent()`. An event with no
`start` is **all-day**: month view drops the time prefix, week/day view lists it
in a "כל היום" band above the hour grid, and the event form has an
"אירוע ללא שעה" checkbox.

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

## Change log (audit trail)

Every insert/update/delete on `events` is mirrored as one row (מתי / מי / מה)
into a Google Sheet (`18321eSScEPEn65TmN7bEODcFqyZqVv_3qz8ebVS-Qtw`, tab
`יומן שינויים`) — **entirely server-side**, so the browser never needs Google
Sheets OAuth scopes (deliberately avoided — that scope is exactly what was
removed earlier to get out of Google's verification requirement; re-adding it
client-side would bring that back for every sign-in).

- **`public.log_event_change()`** — a `security definer` trigger function on
  `events` (`AFTER INSERT OR UPDATE OR DELETE ... FOR EACH ROW`). Builds a
  Hebrew description (prioritizing a date/time change if that's what changed,
  else title, else location, else a generic "updated") and fires it off with
  `net.http_post` (the `pg_net` extension) to a Google Apps Script Web App
  URL, as `{secret, when, who, what}` JSON. `who` comes from `auth.jwt() ->>
  'email'` — same mechanism `is_editor()` relies on.
- The Apps Script (`doPost`, deployed as a Web App, "Execute as: Me" / "Anyone"
  can call it) checks a shared-secret string before appending a row — that
  secret is the *only* thing gating the endpoint, since Apps Script Web Apps
  can't do Google OAuth-in from Postgres. Both the secret and the webhook URL
  are hardcoded into the trigger function's SQL body (safe: PostgREST doesn't
  expose `pg_proc`/function source to anon or authenticated roles — the only
  way to read them is the Supabase SQL editor, which needs the owner's own
  login).
- `net.http_post` is fire-and-forget — a webhook failure never blocks or
  fails the actual calendar write. If a log entry seems to be missing, check
  the `net._http_response` table in the SQL editor for the delivery attempt.
- A bulk operation (recurring add, or an edit/delete scoped to "following"/
  "all") fires the trigger once per affected row, so it logs one line per
  occurrence — that's intentional, not a bug.

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
6. `isEditor()` in `index.html` = `authState.signedIn && authState.canEdit`.
   `canEdit` is set by `refreshCanEdit()` after sign-in, which reads the caller's
   own `editors` row (allowed by the "read own editor row" policy). It's a UI
   convenience so non-editors see "צפייה בלבד" immediately; the RLS write policies
   on `events` are still the real enforcement.
7. **Recurring events have no series table/RRULE — `series_id` is just a shared
   grouping tag** on ordinary rows (see `docs/architecture.md` → "Recurring
   events"). A single-row edit (`editPayload`) and a bulk edit
   (`seriesEditPayload`) both deliberately omit `series_id` from the UPDATE, so
   editing "just this event" can never accidentally sever it from its series.
   A bulk edit also omits `date` — every occurrence keeps its own.
8. **In `public.log_event_change()` (the change-log trigger, see below), never
   test `new is not null` / `old is not null` to distinguish INSERT/UPDATE/
   DELETE.** For a composite row value, Postgres defines `IS NOT NULL` as "all
   fields are non-null" — since `series_id` (and other nullable columns) is
   NULL on every normal event, that test was silently false and skipped the
   whole branch, even on an INSERT. Use `tg_op` instead (`'INSERT'`/`'UPDATE'`/
   `'DELETE'`) — it's unambiguous and doesn't depend on column nullability.
   Also: `to_char()` has no overload for the `time` type — use
   `left(x::text, 5)` to get `HH:MM` from a `time` column, not `to_char(x, ...)`.
