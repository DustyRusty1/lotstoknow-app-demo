# Lots To Know — Project Memory

**Read this first.** This file is the source of truth for where the Lots To
Know trial/demo project stands. Keep it current after every work session —
whoever (or whichever Claude) picks this up next should be able to get fully
oriented from this file alone.

Last updated: 2026-09-23

## What this project is

Lots To Know is the public trial/demo version of Auction Eye (Rusty's
production auction-cataloguing SaaS). It lets a prospective client sign up
with just an email, get a 6-digit access code, and try AI photo cataloguing
without ever touching the real Auction Eye product or its data.

**Hard boundary: Auction Eye production code is never touched by this
project's work.** Everything here stays scoped to the two repos below.

## Repos, infra, and how deploys work

- **`lotstoknow-app-demo`** (public) — the actual demo web app. Served at
  `demo.lotstoknow.com`. Cloudflare Worker name: `auctiq-demo-worker`.
  Single file, `src/worker.js` — serves its own frontend HTML as an inline
  string (`FRONTEND_HTML`) plus the backend API routes in the same file.
  `public/index.html` in this repo is **not** used by the deploy — ignore it.
- **`lotstoknow-demo`** (private) — the landing page + signup flow. Served at
  `lotstoknow.com`. Cloudflare Worker name: `lotstoknow-worker`.
- Both repos are bound to the **same D1 database** — `database_id
  8d390ca9-5886-4caf-a4ac-df06c2ed1b18` — despite being labelled differently
  in each repo's `wrangler.toml` (`auctiq-demo-db` vs `lotstoknow-demo-db`).
  Binding is by ID, so this is fine, just confusing to read.
- Shared table: `demo_tenants` (email, demo_code, ip_address, created_at,
  expires_at, used). Landing page inserts on signup; the demo app only reads.
- **Deploy pipeline**: push to `main` → GitHub Action → `npm install -g
  wrangler` → `wrangler deploy`. ~60–90 seconds end to end. No caching layer
  in front of the Worker (main page sends `Cache-Control: no-store`).
- **GitHub access**: Claude has no standing credentials. Rusty generates a
  short-lived fine-grained PAT (Contents: Read/write, scoped to one repo)
  each session and pastes it in chat. Claude pushes only when explicitly
  told to — never proactively.

## Key decisions made

- Product renamed from "Auctiq" to "Lots to Know" (display with a space).
- New user flow: access code → **instructions screen** ("HOW IT WORKS", 5
  steps) → camera/cataloguing screen. Previously went straight from the gate
  into the camera with zero onboarding.
- Restored the **camera + pencil** icon pair from production Auction Eye's
  real UI (was simplified away to a plain "+" in the demo build). Pencil
  opens manual text entry that feeds into the same review/save step as an AI
  identification — not a separate code path.
- Dev-only conveniences added to make Rusty's own testing tolerable (all
  clearly commented `DEV CONVENIENCE` in the source, meant to be stripped or
  reconsidered before real users hit this):
  - **Remember code**: last working access code is saved to `localStorage`
    and auto-submitted on every reload. Has a "Not you? Use a different
    code" link to clear it — added after the remembered code silently kept
    reusing a stale/fake code even after a real signup.
  - **Permanent test code `341352`**: hardcoded in `checkDemoAccess()`,
    bypasses the `demo_tenants` DB lookup entirely. Works on every
    access-gated route (Identify, Save, Delete, Load, Email, etc.).

## What's currently working (verified)

- Signup → access code → instructions screen → camera screen, full flow.
- Auto-deploy pipeline, reliably.
- Manual entry (pencil) flow, tested end-to-end.
- Photo capture UI + camera icon.
- Banner / status-bar layout on a real iPhone (see "Fixed today" below).
- `demo_code` field name now matches between frontend and backend (see
  broken/unresolved history below — this was a real, long-standing bug).
- Dev code `341352` works on every gated route without needing a DB row.

## Fixed today (chronological, all in `lotstoknow-app-demo` unless noted)

1. Added instructions screen between gate and camera screen (`e910bc5`)
2. Removed the now-redundant floating "?" tutorial button/modal (`8a9258e`)
3. Added `Cache-Control: no-store` on the main page (`802c044`)
4. Fixed banner overlapping the iPhone status bar (`a2c6424`)
5. Tried `position:sticky` to stop the banner covering the header (`cf18313`)
   — **this introduced a worse bug**, see below
6. Found sticky caused duplicate/ghosted header rendering on scroll (a WebKit
   standalone-PWA quirk) — reverted to plain static positioning (`069f814`)
7. Forced a minimum 44px status-bar clearance on the banner regardless of
   what `env(safe-area-inset-top)` reports — it was inconsistent between
   screens for reasons never fully root-caused (`bd4a8e1`)
8. Added the "remember code" dev convenience (`9c5502d`)
9. Restored camera + pencil icons, manual entry flow (`c3067da`)
10. **Fixed Identify always failing** — frontend sent `access_code`, backend
    read `demo_code`, so the field never actually arrived. This meant
    Identify had never worked in this demo build until this fix (`a42d972`)
11. Added "Not you? Use a different code" link (`e9c4dbd`)
12. Added permanent dev test code `341352` (`00a976f`)

## Broken / unresolved

- **🔴 Blocking: Anthropic API key invalid.** Once the `demo_code` bug (item
  10 above) was fixed, Identify now reaches the real Claude API call — and
  fails with `Invalid x-api-key`. The key lives in a GitHub Actions secret
  (`ANTHROPIC_API_KEY`) on `lotstoknow-app-demo`, swapped into
  `wrangler.toml` at deploy time. Claude cannot read or verify GitHub
  secrets (no permission on the PAT). **Rusty needs to check/replace this
  secret** — GitHub repo → Settings → Secrets and variables → Actions —
  with a current key from the Anthropic Console. This is the single thing
  blocking any further testing of the AI identification feature.
- **Unconfirmed: page-duplication/"shading" bug on scroll.** Reported once
  before the sticky-position revert (item 6 above); Rusty said to leave it
  and keep moving, so it was never explicitly re-tested after the fix.
  Worth a dedicated look next session — may already be resolved as a
  side-effect of dropping `position:sticky`, may not be.
- **TEST MODE bypass still live** in `/auth/check` — accepts *any* 6-digit
  numeric code without touching the database. Flagged since the start of
  this project as needing removal before going live. Currently useful for
  testing (and is what caused the whole "invalid or expired" confusion
  earlier today, since it made the gate screen alone meaningless as a
  validity check) but must go before real users arrive.
- **Real email sending unverified.** `POST /email/send` was flagged at the
  very start of this project as returning `{ok:true}` without actually
  sending anything via Resend. Not touched or re-checked today — status
  unknown, assume still broken until verified.
- **Abuse prevention incomplete.** One-code-per-email and IP rate limiting
  (3/24h) are in place. Disposable-email blocking was flagged as a possible
  future TODO, not implemented.

## Next steps, roughly in order

1. Fix the `ANTHROPIC_API_KEY` GitHub secret (blocking everything else)
2. Re-test Identify end-to-end once the key's sorted
3. Dedicated re-check of the scroll/duplication bug
4. Verify real email sending actually works end-to-end
5. Before any real user sees this: remove TEST MODE gate bypass, and decide
   what to do with the two dev-only conveniences (remember-code + `341352`)
   — strip them, or keep remember-code as a real feature and drop the
   hardcoded test code
6. Revisit disposable-email / abuse prevention if it becomes a problem
