# Architecture

Everything lives in one file, `index.html`, structured as: markup → Tailwind/CSS →
one big `<script>` block (an IIFE). There is no bundler — what you see in the file
is what ships.

Two external scripts load in `<head>` before the app runs:

- `https://cdn.tailwindcss.com` — styling (harmless prod warning in console).
- `https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.js` —
  exposes `window.supabase`.

## Config

```js
var SUPABASE_URL = 'https://slsfqlgtthqgkiysywdz.supabase.co';
var SUPABASE_KEY = 'sb_publishable_...';          // publishable (anon) key — public by design
var sb = supabase.createClient(SUPABASE_URL, SUPABASE_KEY);
```

`LOCATIONS` is a small hardcoded array (`{ id, name, color }`) — the location name
is its own id: בית העם, בית אופיר, חורשת נועם, מגרש, דשא מרכזי, מועדון, בית כנסת,
אחר. Edit the array to add/rename/recolour a location. The special `אחר` entry
makes the event form reveal a free-text field (`#evtLocationOther`); an event
saved that way stores the typed string as its `location` and renders with `אחר`'s
colour (`locationById()` synthesises an entry for any unknown id).

## All-day events

An event whose `start_time`/`end_time` are NULL is all-day. `minutesOf('')`
returns `-1` so it sorts first. Month view shows the chip with no time prefix;
`renderTimeGrid` splits each day's events into timed (placed on the hour grid by
`layoutDayEvents`) and all-day (rendered in a "כל היום" band pinned under the day
headers). The form's `#evtAllDay` checkbox hides/clears `#timeFields`;
`eventPayload` sends `null` for an empty time.

## Auth (Supabase Auth + Google provider)

- State object: `var authState = { signedIn: false, email: null, name: null, picture: null };`
  — a local mirror of the Supabase session's `user`, used only for the header UI
  and the `isEditor()` gate.
- `signIn()` → `sb.auth.signInWithOAuth({ provider: 'google', options: { redirectTo: location.origin + location.pathname } })`.
  This is a **full-page redirect** to Google and back (not a popup); on return,
  supabase-js parses the URL and raises a `SIGNED_IN` event.
- `signOut()` → `sb.auth.signOut()` then `clearSession()`.
- `applySession(session, announce)` — fills `authState` from `session.user`
  (`email`, `user_metadata.full_name` / `.name`, `user_metadata.avatar_url` /
  `.picture`), re-renders, and toasts "מחובר/ת בתור…" when `announce` is true.
- `clearSession()` — resets `authState` and re-renders.
- On startup (INIT block): `sb.auth.getSession()` picks up an existing session, and
  `sb.auth.onAuthStateChange((event, session) => …)` handles the sign-in redirect
  landing, the hourly token refresh, and sign-out (here or in another tab).
  `announce` is passed `event === 'SIGNED_IN'` so a page load with a stored session
  doesn't toast.
- `isEditor()` → `return !!authState.signedIn;` — optimistic UI gate only. Real
  enforcement is the RLS write policy (see Data layer).
- `handleWriteError(err)` — Postgres `42501` or HTTP `403` → toast "you're not an
  editor"; `401` / `PGRST301` (expired JWT) → `signOut()`; anything else → generic
  retry toast.

## Header UI

- `#authArea` → `#syncStatus`, plus:
  - `#authSignedOut` → `#signInBtn` (Google "G" SVG + "התחברות עם Google")
  - `#authSignedIn` (hidden by default) → `#authAvatar` (img), `#authName` (span),
    `#signOutBtn` ("התנתקות")
- `renderAuthUI()` toggles those two blocks and fills avatar/name.
- `#brandLogo` (`RamatZviLogo.png`) is absolutely positioned in the header's
  top-left corner.

## Data layer (Supabase `events` table)

Table `events`, columns `id, title, date, start_time, end_time, location,
description, updated_at, created_by`. RLS: `public read` (anon `SELECT`),
`editors insert/update/delete` (gated by `public.is_editor()` against the
`editors` table). See `CLAUDE.md` → "Supabase schema".

- `loadEvents()` — `sb.from('events').select('id,title,date,start:start_time,end:end_time,location,description').order('date').order('start_time')`.
  The `start:start_time` / `end:end_time` aliases keep the rest of the app on
  `start` / `end`. `rowToEvent()` maps each row and trims Postgres `time`
  (`HH:MM:SS`) to `HH:MM`. On error, falls back to `seedEvents()` **only if**
  `state.events` is still empty, then toasts.
- `eventPayload(data)` — maps a form object to the DB column names, stamps
  `updated_at`.
- `insertEvent(data)` / `updateEvent(id, data)` / `removeEvent(id)` — thin wrappers
  over `sb.from('events').insert / .update().eq('id',id) / .delete().eq('id',id)`,
  each re-throwing `res.error`. No row-number bookkeeping (unlike the old Sheets
  version) — the `id` is the key.
- Form submit handler calls `insertEvent` or `updateEvent`, then re-runs
  `loadEvents()` to refresh from the source of truth; gated by `isEditor()`.
- `deleteEvent(id)` calls `removeEvent(id)` then `loadEvents()`; also gated by
  `isEditor()`.
- On load: `state.events = seedEvents()` (instant paint) → `setView('month')` →
  `loadEvents()` (replaces with real data — an empty table just renders an empty
  calendar) → `setInterval(loadEvents, 30000)` (light polling so other people's
  edits show up without a manual refresh). Realtime subscriptions are available in
  Supabase but not used yet — polling is enough.

## Seed / fallback data

`seedEvents()` returns ~16 sample Ramat Zvi events. It's what renders for the
split second before the first `loadEvents()` resolves, and the fallback if
Supabase is ever unreachable *and* nothing has loaded yet.
