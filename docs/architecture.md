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
השכרת ציוד, אחר. Edit the array to add/rename/recolour a location. The special `אחר` entry
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

- State object: `var authState = { signedIn, email, name, picture, canEdit };`
  — a local mirror of the Supabase session's `user`, used only for the header UI
  and the `isEditor()` gate. `canEdit` starts `false` on every sign-in and is
  filled in asynchronously (see below).
- `signIn()` → `sb.auth.signInWithOAuth({ provider: 'google', options: { redirectTo: location.origin + location.pathname } })`.
  This is a **full-page redirect** to Google and back (not a popup); on return,
  supabase-js parses the URL and raises a `SIGNED_IN` event.
- `signOut()` → `sb.auth.signOut()` then `clearSession()`.
- `applySession(session, announce)` — fills `authState` from `session.user`
  (`email`, `user_metadata.full_name` / `.name`, `user_metadata.avatar_url` /
  `.picture`), re-renders, then calls `refreshCanEdit(announce)`.
- `refreshCanEdit(announce)` — `sb.from('editors').select('email')`. A SELECT
  policy lets an authenticated user read *only their own* `editors` row, so a
  non-empty result means "you're an editor". Sets `authState.canEdit`,
  re-renders, and (if `announce`) toasts the right message for either case.
- `clearSession()` — resets `authState` (`canEdit: false`) and re-renders.
- On startup (INIT block): `sb.auth.getSession()` picks up an existing session, and
  `sb.auth.onAuthStateChange((event, session) => …)` handles the sign-in redirect
  landing, the hourly token refresh, and sign-out (here or in another tab).
  `announce` is passed `event === 'SIGNED_IN'` so a page load with a stored session
  doesn't toast.
- `isEditor()` → `return !!(authState.signedIn && authState.canEdit);` — a UI
  convenience so non-editors see "צפייה בלבד" and no add/edit/delete controls
  right away, instead of only finding out on a failed save. The RLS write
  policies on `events` are still the real enforcement.
- `handleWriteError(err)` — Postgres `42501` or HTTP `403` → toast "you're not an
  editor"; `401` / `PGRST301` (expired JWT) → `signOut()`; anything else → generic
  retry toast.

## Header UI

- `#authArea` → `#syncStatus`, plus:
  - `#authSignedOut` → a static "צפייה בלבד" badge + `#signInBtn` (Google "G"
    SVG + "התחברות עם Google") — a guest is inherently view-only, so this badge
    needs no JS.
  - `#authSignedIn` (hidden by default) → `#authAvatar` (img), `#authName`
    (span), `#authViewOnly` (badge, shown when signed in but not an editor),
    `#signOutBtn` ("התנתקות")
- `renderAuthUI()` toggles those two blocks, fills avatar/name/badge, and also
  toggles `#addEventBtn`'s `hidden` (`!isEditor()`) — it's the one place that
  reacts to every auth state change.
- `#brandLogo` (`RamatZviLogo.png`) is absolutely positioned in the header's
  top-left corner. `#addEventBtn` sits in the header's top row next to the view
  tabs (physically to their left in this RTL layout).

## Day selection + adding an event

Clicking a day (a month cell, a week/day column header, or an hour slot in
week/day view) never opens the add-event form directly — it only calls
`selectDate(dateKey, time)`, which sets `state.selectedDate`/`selectedTime` and
re-renders. The selected day gets a yellow highlight (`.is-selected` on
`.day-cell`/`.day-num` — same pattern as `.is-today`, just yellow instead of
accent-blue; declared after `.is-today` in the stylesheet so a day that's both
today and selected reads as selected). Any visitor can select a day — it's
purely a visual focus, not an authorization check.

The only way to open the add-event form is `#addEventBtn` (hidden unless
`isEditor()`), in the header next to the view tabs. Its click handler calls
`openAddModal(state.selectedDate || toKey(state.cursor), state.selectedTime)` —
it defaults to whatever's selected, or the current view's date if nothing is.

`findConflict(data, excludeId)` runs on form submit (add *and* edit): same
`date` + `location` + overlapping hours as another event in `state.events`
(skipping `excludeId` so editing an event doesn't conflict with itself).
`timeRange(ev)` treats an all-day event (no start/end) as spanning
00:00–24:00, so it conflicts with *any* event at that location that day,
timed or all-day. A hit shows `window.confirm(...)`; declining aborts the
save before it reaches Supabase.

## Recurring events (add only)

There is no recurrence "series" concept in the schema — a recurring event is
**materialized as N independent rows** at add time, one per occurrence, each
editable/deletable on its own afterwards (no series linkage, no "edit all
occurrences"). This is deliberate: it fits the existing flat `events` table
and rendering with zero changes, at the cost of Google-Calendar-style series
editing. `openEditModal` hides `#recurringSection` entirely, so this only
ever applies on add.

- `#evtRecurring` reveals `#evtRecurFreq` (daily/weekly/monthly/yearly) and
  `#evtRecurUntil` (an inclusive end date, required).
- `buildRecurrenceDates(startDate, untilDate, freq)` returns the date-key
  list, capped at `RECUR_MAX` (366) as a safety net against a runaway range.
  monthly/yearly clamp to the target month's last day (so "31 Jan monthly"
  lands on 28 Feb, 31 Mar, 30 Apr, ... — not JS's raw `setMonth` overflow
  into the next month).
- Each date is conflict-checked (`findConflict`) against *existing* events
  only (occurrences never conflict with each other, since the generator
  never repeats a date); a single `confirm()` summarizes the count if any
  hit, instead of one prompt per occurrence.
- `insertEvents(dataArray)` sends all occurrences as **one** multi-row
  `INSERT`, not N separate requests.

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
