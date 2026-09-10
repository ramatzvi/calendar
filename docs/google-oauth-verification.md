> **⚠️ SUPERSEDED — kept only as history.** The app was migrated off the Google
> Sheets datastore to Supabase, and the sensitive `.../auth/spreadsheets` scope
> was removed. With only the non-sensitive `email` / `profile` / `openid` scopes,
> Google verification is **not required** and there is no "unverified app"
> warning. Nothing in this document needs to be acted on. The one remaining
> cleanup task (remove the sensitive scope from the Google consent screen) is
> tracked in the migration plan, not here.

---

# Google OAuth app verification — status & how to continue

**Goal (user's explicit request):** *"open sign-in to everyone (via Google's verification process)"* — let any Google user sign in without seeing Google's "unverified app" warning, by completing Google's formal OAuth verification for the app in Google Cloud project **`ramat-zvi-calendar`**.

Google Cloud console: https://console.cloud.google.com/auth/overview?project=ramat-zvi-calendar

## Background Google requires for this

- Publishing status must be **"In production"** (not "Testing") for verification to matter — Testing mode caps sign-in to a 100-user allowlist regardless of verification. → **Done.**
- Because the app requests a *sensitive* scope (`https://www.googleapis.com/auth/spreadsheets`), Google verification has **two sequential tracks**:
  1. **Branding verification** — validates app name, logo, home page, privacy policy, ToS, and that the homepage domain is provably owned by the developer (via Search Console). Must pass before track 2 unlocks.
  2. **Data access (scope) verification** — justifies why the sensitive Sheets scope is needed. Gated behind branding verification succeeding.
- After both, the app is submitted for **Google's manual review**, which is on Google's own timeline (commonly days), not something resolvable from the console alone.

## What's been completed

| Item | Status |
|---|---|
| Publishing status | **In production** (was "Testing") |
| App name / support email | Filled in Branding |
| Application home page | `https://ramatzvi.github.io/calendar/` |
| Application privacy policy link | `https://ramatzvi.github.io/calendar/privacy.html` — saved |
| Application Terms of Service link | `https://ramatzvi.github.io/calendar/terms.html` — saved |
| Authorised domain | `ramatzvi.github.io` — registered |
| Scopes registered under Data access | `.../auth/userinfo.email`, `.../auth/userinfo.profile` (non-sensitive), `https://www.googleapis.com/auth/spreadsheets` (sensitive) — saved |
| Domain ownership | Verified in **Google Search Console** for property `https://ramatzvi.github.io/calendar/` (URL-prefix property, HTML-file verification method) — confirmed "Ownership verified" |

## What's currently blocked

Branding re-verification (Google Cloud console → **Google Auth Platform → Branding** → "Verification status" panel → **View issues** → select **"I have fixed the issues"** → **Proceed**) has been retried twice. Both times it completes quickly (well under the stated "up to 5 minutes") and returns to the **same** error:

> The website of your homepage URL 'https://ramatzvi.github.io/calendar/' is not registered to you.

This is despite:
- Search Console showing that exact URL as "Ownership verified" under the same signed-in Google account used in Cloud Console.
- The Branding "Application home page" field matching the verified Search Console property **exactly** (`https://ramatzvi.github.io/calendar/`).
- The authorised domain (`ramatzvi.github.io`) being correctly registered.

**Conclusion:** this is very likely a propagation delay between Search Console's verification record and Google's OAuth branding verifier — a known real-world lag (sometimes well beyond the "5 minutes" the UI states) rather than a configuration mistake. There is nothing left to *configure* here; it just needs to be retried after waiting.

## Exact next steps

1. Go to https://console.cloud.google.com/auth/branding?project=ramat-zvi-calendar
2. Under "Verification status," click **View issues**.
3. If it still shows the same "homepage URL not registered to you" issue, wait (try again in a few hours if a recent retry just failed — don't hammer it every few seconds) and retry: select **"I have fixed the issues"** → **Proceed**.
4. If it shows a **different** issue, diagnose that specific issue (it will describe what's missing).
5. Once branding verification shows a passing/verified state, go to **Verification centre** (https://console.cloud.google.com/auth/verification?project=ramat-zvi-calendar) and click **"Prepare for verification"** to start the **Data access (sensitive scope)** track for `https://www.googleapis.com/auth/spreadsheets`. This step has not been explored in detail yet since it was gated behind branding — expect it to ask for a written justification of why the app needs Sheets write access, and possibly a demo video of the OAuth consent flow in the app (Google's typical requirement for sensitive-scope verification; confirm exact requirements when you reach this screen).
6. Submit for Google's final review. This is a manual review on Google's side — budget for it taking real days, not something to keep retrying from the console.

## Troubleshooting notes from the first pass (useful if new issues appear)

- **`Error 400: invalid_request ... doesn't comply with Google's OAuth 2.0 policy`** at actual sign-in time (as opposed to the verification checker) had two distinct causes, both already fixed in the shipped code — see CLAUDE.md "Known gotchas." If this error reappears, check those two things first (an `openid` scope leaking back in, or a scope quietly becoming unregistered under Data access) before assuming it's a new bug.
- Google Search Console: used **URL prefix** property type, not **Domain** type — Domain-type verification needs DNS TXT records, which aren't practical to control for a GitHub Pages subpath (`ramatzvi.github.io/calendar/` is a subpath of a domain — `github.io` — that Ofir doesn't own outright). Verification method used: **HTML file** (`google2d2eb4e31d5b5e0c.html` at the site root, containing exactly `google-site-verification: google2d2eb4e31d5b5e0c.html`). Do not remove that file from the repo, or delete the Search Console property, without re-verifying branding again afterward.
