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
  - `#logLink` lives in the header's top row, not the auth row — a small
    icon-only square button (document/log glyph) styled like `#addEventBtn`
    (`btn-primary`). Both buttons are wrapped together in one
    `<div class="... ms-auto">` right after the view-tabs, placed before
    `#addEventBtn` in the DOM so in this RTL layout `#logLink` renders just
    to its right; `ms-auto` pins the pair to the trailing (left) edge of
    whichever flex-wrap line they land on, instead of relying on the
    parent's `justify-between` to do it (which only worked by coincidence
    for some viewport widths — see git history if this regresses). On
    narrow/mobile widths `#addEventBtn`'s label collapses to icon-only
    (`<span class="hidden sm:inline">`) and its box shrinks to the same
    `w-8 h-8` square as `#logLink` (`sm:w-auto sm:h-auto` restores the
    labeled size at the `sm` breakpoint) so the two stay the same size.
    Shown only for `canViewLog()` — see "Change log viewer" below.
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

## Recurring events

There is no RRULE/virtual-occurrence engine — a recurring event is
**materialized as N independent rows** at add time, one per occurrence, all
sharing a client-generated `series_id` (`crypto.randomUUID()`; plain `uuid`
column on `events`, no FK, no separate `series` table, no extra RLS — a bulk
`UPDATE`/`DELETE ... WHERE series_id = X` is already covered by the existing
`editors` policies). Each occurrence is still a completely normal `events`
row otherwise, so all existing rendering/query code needed zero changes.

**Add** (`openAddModal`/submit handler): `#evtRecurring` reveals
`#evtRecurFreq` (daily/weekly/monthly/yearly) and `#evtRecurUntil` (an
inclusive end date, required).
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
  `INSERT`, tagging every row with the same new `series_id`.

**Edit/delete a series member** (`renderViewFooter`, gated on `ev.seriesId`):
clicking עריכה/מחיקה shows a "רק אירוע זה / אירוע זה וכל הבאים / כל האירועים
בסדרה" scope chooser (skipped entirely for a non-series event — same
single-row flow as before). The chosen scope drives:
- **this** → the existing single-row `updateEvent`/`removeEvent`, unchanged.
- **following** → `updateSeriesFrom`/`removeSeriesFrom` (`WHERE series_id = X
  AND date >= <the date this event had when the modal opened>`).
- **all** → `updateSeriesAll`/`removeSeriesAll` (`WHERE series_id = X`).

`#evtDate` stays editable for a bulk edit too, but changing it **shifts**
every affected occurrence by the same number of days rather than collapsing
them onto one date (`#dateBulkHint` explains this). The submit handler
computes `deltaDays` from `data.date` vs. `state.editOriginalDate`
(the event's date when the modal opened); if it's non-zero, `shiftSeriesDates`
runs instead of the plain `updateSeriesFrom`/`updateSeriesAll`:
it re-fetches the affected rows' own dates, adds `deltaDays` to each, and
writes them all back with **one `.upsert()`** (not `.update()` — Supabase's
`update()` can only set every matched row to the *same* literal value, so a
per-row "add N days to your own date" needs `upsert`'s per-row payload
instead; RLS applies exactly as it would to a normal insert/update, no
special-casing needed). `seriesEditPayload` still strips `date` from the
non-shift path, and also `series_id` — so a bulk field-only edit never
touches either. `editPayload` (single-row update) also strips `series_id`,
so editing "just this event" never severs it from its series by accident.
Bulk edits (shifted or not) skip `findConflict` — checking one edit against
a whole series' worth of rows is a fuzzier problem than the single-event case,
so it's out of scope.

Adding a recurring event: `#evtRecurUntil` defaults to (and its `min` is
floored at) the event's own date, so the picker opens on the relevant month
instead of today; both stay in sync if `#evtDate` is changed afterward.

## Data layer (Supabase `events` table)

Table `events`, columns `id, title, date, start_time, end_time, location,
description, updated_at, created_by, series_id`. RLS: `public read` (anon
`SELECT`), `editors insert/update/delete` (gated by `public.is_editor()`
against the `editors` table) — these already cover bulk series operations,
since they don't key off row content. See `CLAUDE.md` → "Supabase schema".

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
  calendar) → adaptive polling so other people's edits show up without a manual
  refresh: `scheduleNextPoll()` self-reschedules via `setTimeout` (not
  `setInterval`) at `pollDelayMs()` — 120s during the day (07:00–21:59, by each
  viewer's own local clock) or 600s at night (22:00–06:59) — and a
  `visibilitychange` listener clears the pending timer while the tab is hidden
  (`document.hidden`) and fires an immediate `loadEvents()` + resumes the cycle
  the moment it's visible again, instead of leaving a backgrounded tab polling
  uselessly or making the user wait out a stale interval on return. Realtime
  subscriptions are available in Supabase but not used yet — polling is enough.

## Change log viewer ("קובץ לוג")

`LOG_VIEWERS` (`['adi.landshaft@gmail.com', 'ofir.landshaft@gmail.com']`) is
declared right after the Supabase client. `canViewLog()` is just
`authState.signedIn && LOG_VIEWERS.indexOf(authState.email) !== -1` — a UI
convenience that hides `#logLink`; the real gate is Postgres RLS (see below).

This opens as a **real separate popup window**, not an in-page modal (the
user asked for it to feel like its own full document, not a small dialog).
`openLogModal()` calls `window.open('', 'ramatZviChangeLog', ...)`
**synchronously** inside the click handler — before any `fetch`/`sb` call —
so browsers don't treat it as a blocked popup; `LOG_WINDOW_HTML` (a
self-contained HTML string with its own `<style>`, no Tailwind) is written
into it via `document.write()`, then filled in asynchronously. Clicking the
link again while the window is still open just `.focus()`es it and
re-fetches, instead of opening a second one (same `logWin` reference, same
window name).

- `fetchChangeLog()` is a plain `sb.from('change_log').select('happened_at,who,what').order('happened_at', {ascending: true})`
  (oldest first) — RLS (`public.is_log_viewer()`) restricts the rows to only
  what a log viewer is allowed to see; there's no Apps Script call involved
  at all.
- `clearChangeLogRequest()` is `sb.rpc('clear_change_log')`, a Postgres RPC
  that re-checks `is_log_viewer()` itself before deleting — a client can't
  spoof this by editing `index.html`, same as every other RLS-gated action
  in this app. See `CLAUDE.md` → "Viewing/clearing the log from the app" for
  why this doesn't go through Apps Script (a dead end: anonymous-access Web
  Apps can't get `UrlFetchApp` authorization, confirmed after extensive
  testing).
- `renderLogTable()`/`renderLogFooter()` render into `logWin.document`
  (not the main page's `document`) — every DOM call in this feature is
  guarded by `if (!logWin || logWin.closed) return;` since the popup can be
  closed by the user at any point, including mid-fetch.
- `renderLogFooter()` toggles between the normal footer (מחק שינויים /
  סגירה, the latter just `logWin.close()`) and a confirm-before-delete
  footer (ביטול / אישור מחיקה) — the
  same two-step pattern `renderViewFooter()` uses for deleting an event.
  Confirming calls `clearChangeLogRequest()` then reloads the table.

## Seed / fallback data

`seedEvents()` returns ~16 sample Ramat Zvi events. It's what renders for the
split second before the first `loadEvents()` resolves, and the fallback if
Supabase is ever unreachable *and* nothing has loaded yet.
