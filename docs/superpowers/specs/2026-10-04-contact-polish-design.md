# Contact Polish — Design Spec (Approach A)

Date: 2026-10-04
Scope: Sub-project 1 of 4 (Contact polish → Projects manager → Blog/updates → Visual polish)
Stack constraint: basic HTML + CSS + JavaScript only. No React, no Vite, no build step, no new npm deps.
Source of truth: `Final_portfolio/` (`index.html`, `styles-2.css`, `app-2.js`, `login.html`, `admin.html`)
Status: Approved design, awaiting spec review before implementation plan.

## 1. Context

Current contact flow (`Final_portfolio/index.html:1000-1046`, handler `Final_portfolio/app-2.js:464-504`):

- Form fields: name, email, message. Uses native `checkValidity()` + `reportValidity()`.
- Dual-write on submit: POST to Web3Forms (`access_key` hardcoded at `app-2.js:480`) for email forwarding, then insert into Supabase `messages` table (`sbqxizjtgdfixvnwtbry.supabase.co`, publishable key at `app-2.js:491`) for `admin.html` inbox.
- No honeypot, no time-trap, no rate-limit, no length caps, sequential awaits, generic fallback message, button states minimal.
- Inbox (`admin.html:182-189`) and login (`login.html:31-37`) already work and must not regress.

Goal: harden spam + validation + UX + error handling without changing backends, without migrations, keeping email forwarding working.

Non-goals: no Supabase RLS change, no dropping Web3Forms, no offline outbox queue, no CAPTCHA, no server code, no changes to `portfolio_nextjs/` or root Vite app.

## 2. Architecture

100% static. Three files change:

- `Final_portfolio/index.html` — form markup only.
- `Final_portfolio/app-2.js` — contact handler only (IIFE at `app-2.js:395-514`).
- `Final_portfolio/styles-2.css` — error/honeypot styles only (additive, ~20 lines).

No new files, no new CDN, no config files. Works from `file://`, `python -m http.server`, or any static host. Existing GSAP/Lenis/Supabase CDN tags untouched.

## 3. Components

### 3.1 HTML (`index.html` contact form)

- Keep `id="contactForm"`, `novalidate`, `id="cfName"`, `id="cfEmail"`, `id="cfMsg"`, `id="cfSend"`, `id="cfOk"`.
- Add `maxlength`: name 100, email 254, message 2000. Add `minlength="2"` on name/message.
- Add honeypot: `<input id="cfCompany" name="company" tabindex="-1" autocomplete="off" aria-hidden="true">` inside form. Hidden via CSS, not `type="hidden"`.
- Add `data-started-at` set on page load via JS (not in HTML) for time-trap.
- Add per-field error slots: `<p class="field-error" id="cfNameErr" hidden>` etc., wired with `aria-describedby`.
- Add `id="cfErr"` for form-level network errors (distinct from `id="cfOk"` success note).
- Both copy-email buttons keep existing IDs (duplicate `id="copyEmail"` exists at `index.html:868,1038` — do not rename in this spec; handler uses `querySelector` first match, out of scope).

### 3.2 CSS (`styles-2.css`, additive)

- `.hp, #cfCompany { position:absolute !important; left:-9999px !important; top:auto !important; width:1px !important; height:1px !important; opacity:0 !important; pointer-events:none !important; }`
- `.field-error { font-size:12px; color:#f87171; margin-top:6px; }` plus light-theme variant `html[data-theme="light"] .field-error { color:#b91c1c; }`.
- `#cfErr { color:#f87171; }` and `#cfErr[hidden], #cfOk[hidden], .field-error[hidden] { display:none !important; }`
- `input[aria-invalid="true"], textarea[aria-invalid="true"] { border-color:#f87171; }`
- No layout changes, no breakpoint changes.

### 3.3 JS (`app-2.js` contact IIFE)

Replace sequential try/catch with:

- Constants at top of contact block: `MAXLEN = {name:100, email:254, message:2000}`, `MIN_TIME_MS=3000`, `RATE_LIMIT_MS=60000`, `TIMEOUT_MS=10000`, `LS_KEY="v2-last-send"`.
- On load: set `form.dataset.startedAt = Date.now()`.
- Helpers: `trim()`, `setErr(input, errEl, msg)`, `clearErrs()`, `isBot()` (honeypot non-empty), `isTooFast()` (now - startedAt < 3000), `isRateLimited()` (now - LS last < 60000).
- Validate: required, `type=email` validity, length caps. On fail: show inline errors, `aria-invalid`, focus first invalid, return early with no fetch.
- Bot path: if honeypot filled or too fast and not obviously human double-submit, silently fake success (`okNote("Message sent!...")`, reset, no network). Rate-limit path: human-readable `Please wait a minute before sending again.` No alert().
- Send: disable `sendBtn`, text `Sending…`. Use `AbortController` with 10s timeout per request. Fire Web3Forms POST and Supabase insert in parallel via `Promise.allSettled`.
- Web3Forms payload unchanged (same `access_key`, `subject`, `from_name`) to preserve email delivery.
- Supabase insert unchanged (`supabase.createClient(URL, KEY).from("messages").insert({name,email,message})`).
- Success rule: if either settles fulfilled with success flag, treat as success: `okNote("Message sent! I'll get back to you within a day.")`, `cf.reset()`, reset `startedAt`, set `LS last`. Hide `cfErr`.
- Double-fail: `errNote("Couldn't send just now — email me directly at j.doruca109@gmail.com.")`, keep field values, re-enable, focus send button. No `alert()`, no `mailto:` auto-open.
- Always re-enable button in `finally`, restore text `Send message`.
- Copy-email handler unchanged.

## 4. Data Flow

1. User types → native + JS validation on submit only (no on-input spam).
2. Bot checks (honeypot, time-trap, rate-limit) → early return.
3. Disable UI → parallel `allSettled([web3forms, supabase])` with timeout.
4. Either ok → success UI + reset + timestamp. Both fail → error UI + preserve draft.

No ordering dependency between email and DB. No retry loop. No queue persistence.

## 5. Error Handling

- Validation errors: inline per-field, `aria-live="polite"` via existing `reportValidity` fallback, focus management.
- Network abort/timeout: caught per-promise, surfaced only if both fail.
- Web3Forms `{success:false}` or non-OK HTTP: treated as failure for that branch.
- Supabase `{error}`: treated as failure for that branch, message not shown raw to user (avoid leaking RLS text); logged to console only.
- JSON parse failure: caught, branch fails.
- Supabase JS CDN missing (`window.supabase` undefined): skip DB branch, rely on email branch.
- Reduced-motion / no-JS: form still has `required` + `type=email`; without JS no send (acceptable, current behavior).

## 6. Security / Secrets

- No new secrets. Existing `access_key` and Supabase publishable key remain in frontend (unavoidable for static). Mitigation: restrict Web3Forms key to portfolio domain in Web3Forms dashboard; confirm Supabase `messages` RLS is anon-insert-only, no anon-select (verified via `admin.html` authenticated read still working).
- Recovery codes (`recovery-codes.txt`) are out of scope for code — owner must regenerate in GitHub settings and delete file from Downloads. Never commit `*.txt` codes, `.env`, or inbox data.
- Length caps + trim prevent oversized inserts. Honeypot name `company` is generic to catch bots; do not log honeypot values.

## 7. Testing (manual, basic only)

Serve: `python -m http.server` from `Final_portfolio/` (or VS Code Live Server). No build.

- Happy path: valid name/email/message → success note, fields clear, new row in `admin.html` inbox, email received via Web3Forms.
- Validation: empty submit → inline errors; bad email → error; 2001-char message blocked by `maxlength` + JS cap; focus moves to first error.
- Bot: fill honeypot via devtools → fake success, no network; submit within 3s → fake success or rate message; second submit within 60s → rate message.
- Resilience: block `api.web3forms.com` (devtools) → still success via Supabase; block Supabase → still success via email; block both → error note, draft preserved.
- Regression: theme toggle, mobile menu, GSAP reveals, `toTop`, chat widget, `login.html` → `admin.html` flow unaffected.
- Browsers: latest Chrome + Firefox, 360px mobile width, keyboard-only submit.

Success criteria: all above pass, diff touches only 3 files, no console errors on load, no change to inbox read/update/delete logic.

## 8. YAGNI / Out of Scope

No CAPTCHA/Turnstile, no Edge Function, no RLS SQL, no removing Web3Forms, no localStorage outbox, no character counters, no i18n, no analytics, no changes to Vite/React root or `portfolio_nextjs/`.
