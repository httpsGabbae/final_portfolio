# Contact Polish Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Harden Final_portfolio contact form with spam guards, validation, and resilient dual-write while staying 100% static.

**Architecture:** Edit only `Final_portfolio/index.html`, `Final_portfolio/styles-2.css`, `Final_portfolio/app-2.js`. Keep Web3Forms email + Supabase inbox dual-write, fire in parallel with timeout, fake-success bots, rate-limit humans.

**Tech Stack:** Basic HTML, CSS, JavaScript (no build, no npm, no new CDN). Existing CDNs only: Supabase JS v2, GSAP, Lenis.

**Spec:** `docs/superpowers/specs/2026-10-04-contact-polish-design.md`

## Global Constraints

- Basic HTML/CSS/JS only — no React, no Vite, no build step, no new dependencies.
- Touch only: `Final_portfolio/index.html`, `Final_portfolio/styles-2.css`, `Final_portfolio/app-2.js`.
- Preserve IDs: `contactForm`, `cfName`, `cfEmail`, `cfMsg`, `cfSend`, `cfOk`.
- Length caps: name 100, email 254, message 2000; time-trap 3000ms; rate-limit 60000ms; fetch timeout 10000ms.
- Success if either Web3Forms or Supabase succeeds; error only if both fail.
- Never commit secrets, recovery codes, or inbox data.
- Serve test root is `Final_portfolio/`, not repo root.

---

### Task 1: HTML form markup — honeypot, caps, error slots

**Files:**
- Modify: `Final_portfolio/index.html:1000-1046`
- Test: manual browser check via `python -m http.server`

**Interfaces:**
- Consumes: none
- Produces: DOM ids `cfCompany`, `cfNameErr`, `cfEmailErr`, `cfMsgErr`, `cfErr` with attributes Task 2/3 rely on. `cfCompany` is `input[type=text]`, others are `p.field-error[hidden]`.

- [ ] **Step 1: Write the failing check (no code change yet)**

Open current form and confirm missing pieces:

```powershell
python -m http.server 8000
# in workdir Final_portfolio/
```

Manual: open `http://localhost:8000/index.html#contact`, DevTools Elements search `#cfCompany` → expect 0 matches. Search `#cfErr` → 0 matches. `document.getElementById('cfName').hasAttribute('maxlength')` → `false`. This is FAIL baseline.

Run: `python -m http.server 8000` in `Final_portfolio/`
Expected: FAIL — ids missing, no maxlength.

- [ ] **Step 2: Run check to verify it fails**

Run: open page, run in console:

```js
!!document.getElementById('cfCompany') && !!document.getElementById('cfErr')
```

Expected: `false`

- [ ] **Step 3: Write minimal markup**

Replace `Final_portfolio/index.html:1000-1046` inner form with:

```html
<form class="contact-form" id="contactForm" novalidate>
  <div class="frow">
    <div>
      <label class="mono" for="cfName">Name</label><input
        id="cfName"
        name="name"
        required
        minlength="2"
        maxlength="100"
        placeholder="Your name"
        autocomplete="name"
        aria-describedby="cfNameErr"
      />
      <p class="field-error" id="cfNameErr" hidden></p>
    </div>
    <div>
      <label class="mono" for="cfEmail">Email</label><input
        id="cfEmail"
        name="email"
        type="email"
        required
        maxlength="254"
        placeholder="you@mail.com"
        autocomplete="email"
        aria-describedby="cfEmailErr"
      />
      <p class="field-error" id="cfEmailErr" hidden></p>
    </div>
  </div>
  <div>
    <label class="mono" for="cfMsg">Message</label><textarea
      id="cfMsg"
      name="message"
      rows="4"
      required
      minlength="2"
      maxlength="2000"
      placeholder="Hi Gabbae, I saw your payroll system and…"
      aria-describedby="cfMsgErr"
    ></textarea>
    <p class="field-error" id="cfMsgErr" hidden></p>
  </div>
  <input id="cfCompany" name="company" type="text" class="hp" tabindex="-1" autocomplete="off" aria-hidden="true" />
  <div class="hero-row">
    <button class="btn btn-solid" id="cfSend" type="submit">
      Send message
    </button>
    <button class="btn" id="copyEmail" type="button">
      Copy email
    </button>
  </div>
  <p class="form-ok" id="cfOk" hidden>
    Message sent! I'll get back to you within a day.
  </p>
  <p class="form-ok form-err" id="cfErr" hidden role="alert"></p>
</form>
```

Do not rename duplicate `id="copyEmail"` elsewhere in this task.

- [ ] **Step 4: Run check to verify it passes**

Run: refresh `http://localhost:8000/index.html#contact`, console:

```js
!!document.getElementById('cfCompany') && !!document.getElementById('cfErr') && document.getElementById('cfName').getAttribute('maxlength')==='100' && document.getElementById('cfMsg').getAttribute('maxlength')==='2000'
```

Expected: `true`. Visual: no visible honeypot field, layout unchanged.

- [ ] **Step 5: Commit**

```bash
git add Final_portfolio/index.html
git commit -m "feat(contact): add honeypot, caps, error slots"
```

---

### Task 2: CSS — honeypot hide + error states (additive only)

**Files:**
- Modify: `Final_portfolio/styles-2.css:968-975` (append after `.form-ok` block, do not edit existing rules)
- Test: manual visual check

**Interfaces:**
- Consumes: DOM classes/ids from Task 1: `.hp`, `#cfCompany`, `.field-error`, `#cfErr`, `.form-err`
- Produces: visual hiding + error styling Task 3 toggles via `hidden` and `aria-invalid`.

- [ ] **Step 1: Write the failing check**

With Task 1 done but Task 2 not applied, honeypot is visible (unstyled). Open `http://localhost:8000/index.html#contact` → see extra text input below message. DevTools `getComputedStyle(document.getElementById('cfCompany')).left` → `auto`, not `-9999px`. FAIL baseline.

Run: visual inspect + console `getComputedStyle(document.getElementById('cfCompany')).position`
Expected: FAIL — `static`, visible.

- [ ] **Step 2: Run check to verify it fails**

Run: same console check.
Expected: FAIL — honeypot not off-screen.

- [ ] **Step 3: Write minimal CSS**

Append after `.form-ok` block at `styles-2.css:968-975`:

```css
.hp,
#cfCompany {
  position: absolute !important;
  left: -9999px !important;
  top: auto !important;
  width: 1px !important;
  height: 1px !important;
  opacity: 0 !important;
  pointer-events: none !important;
}
.field-error {
  margin: 6px 0 0;
  font-size: 12px;
  color: #f87171;
}
html[data-theme="light"] .field-error {
  color: #b91c1c;
}
#cfErr {
  color: #f87171;
}
html[data-theme="light"] #cfErr {
  color: #b91c1c;
}
#cfErr[hidden],
#cfOk[hidden],
.field-error[hidden] {
  display: none !important;
}
.contact-form input[aria-invalid="true"],
.contact-form textarea[aria-invalid="true"] {
  border-color: #f87171;
}
```

No other selectors edited.

- [ ] **Step 4: Run check to verify it passes**

Run: hard refresh, console:

```js
getComputedStyle(document.getElementById('cfCompany')).left === '-9999px' && document.getElementById('cfNameErr').hidden === true
```

Expected: `true`. Toggle theme button → light theme error color still readable. Mobile 360px width: no layout shift.

- [ ] **Step 5: Commit**

```bash
git add Final_portfolio/styles-2.css
git commit -m "feat(contact): hide honeypot, style inline errors"
```

---

### Task 3: JS validation + bot guards (no network change yet)

**Files:**
- Modify: `Final_portfolio/app-2.js:464-474` (contact block header + helpers, keep send logic stubbed for Task 4)
- Test: manual console + form submit with network untouched

**Interfaces:**
- Consumes: DOM from Task 1
- Produces: functions `clearContactErrs()`, `setFieldErr(input, errEl, msg)`, `isBotSubmit()`, `isRateLimited()`; constants `CF_MAX`, `CF_MIN_MS=3000`, `CF_RATE_MS=60000`, `CF_LS="v2-last-send"`. Task 4 calls these exact names.

- [ ] **Step 1: Write the failing check**

Current `app-2.js` has no `isBotSubmit`, no maxlength JS caps. Console `typeof isBotSubmit` → not global (inside IIFE, so test via behavior): fill 2001-char message → still attempts fetch (no client cap). Submit within 2s of load → still fetches. FAIL baseline.

Run: open contact, paste 2500 chars, submit, observe fetch in Network tab.
Expected: FAIL — request sent despite oversize.

- [ ] **Step 2: Run check to verify it fails**

Run: same manual test.
Expected: FAIL — oversize + fast submit not blocked.

- [ ] **Step 3: Write minimal implementation (helpers + validation gate)**

In `app-2.js` contact IIFE, after `const cf = $("#contactForm");` insert:

```js
const CF_MAX = { name: 100, email: 254, message: 2000 };
const CF_MIN_MS = 3000;
const CF_RATE_MS = 60000;
const CF_LS = "v2-last-send";
if (cf && !cf.dataset.startedAt) cf.dataset.startedAt = String(Date.now());

function clearContactErrs() {
  ["cfName", "cfEmail", "cfMsg"].forEach(function (id) {
    const inp = document.getElementById(id);
    const err = document.getElementById(id + "Err");
    if (inp) inp.removeAttribute("aria-invalid");
    if (err) { err.textContent = ""; err.hidden = true; }
  });
  const fe = document.getElementById("cfErr");
  if (fe) { fe.textContent = ""; fe.hidden = true; }
}

function setFieldErr(inputId, msg) {
  const inp = document.getElementById(inputId);
  const err = document.getElementById(inputId + "Err");
  if (inp) inp.setAttribute("aria-invalid", "true");
  if (err) { err.textContent = msg; err.hidden = false; }
}

function isBotSubmit() {
  const hp = document.getElementById("cfCompany");
  if (hp && hp.value) return true;
  const st = cf ? Number(cf.dataset.startedAt || 0) : 0;
  if (st && Date.now() - st < CF_MIN_MS) return true;
  return false;
}

function isRateLimited() {
  try {
    const last = Number(localStorage.getItem(CF_LS) || 0);
    return last && Date.now() - last < CF_RATE_MS;
  } catch (_) { return false; }
}
```

Wire into submit handler top (before existing fetch): call `clearContactErrs()`, trim values, check lengths against `CF_MAX`, check `cf.checkValidity()`, call `setFieldErr` + focus first invalid + `return`. Then `if (isBotSubmit())` fake-success path (set okNote, reset, return without fetch). Then `if (isRateLimited())` show `cfErr` text `Please wait a minute before sending again.` + return. Do not change dual-write yet.

Exact fake-success snippet:

```js
if (isBotSubmit()) {
  const el = document.getElementById("cfOk");
  if (el) { el.textContent = "Message sent! I'll get back to you within a day."; el.hidden = false; }
  cf.reset();
  cf.dataset.startedAt = String(Date.now());
  return;
}
```

- [ ] **Step 4: Run check to verify it passes**

Run: refresh, tests:
```js
// 1. oversize blocked, no fetch
document.getElementById('cfName').value='A'; document.getElementById('cfEmail').value='a@b.co'; document.getElementById('cfMsg').value='x'.repeat(2001); document.getElementById('contactForm').requestSubmit();
// expect #cfMsgErr visible, Network 0 new requests
// 2. honeypot
document.getElementById('cfCompany').value='bot'; document.getElementById('contactForm').requestSubmit();
// expect #cfOk visible, Network 0 new requests
```
Expected: PASS — both blocked client-side.

- [ ] **Step 5: Commit**

```bash
git add Final_portfolio/app-2.js
git commit -m "feat(contact): add validation and bot guards"
```

---

### Task 4: JS parallel dual-write + states + error handling

**Files:**
- Modify: `Final_portfolio/app-2.js:466-504` (replace sequential Web3Forms-then-Supabase with parallel allSettled)
- Test: manual with DevTools request blocking

**Interfaces:**
- Consumes: helpers from Task 3, DOM `cfSend`, `cfOk`, `cfErr`
- Produces: behavior — either-branch success, both-fail error, button disable/restore, `localStorage[CF_LS]` set on success. No new globals except `__contactLast` for debugging (optional).

- [ ] **Step 1: Write the failing check**

Current code awaits Web3Forms then Supabase sequentially, no timeout, success requires both. Block `api.web3forms.com` in DevTools → submit valid → shows error even though Supabase would succeed. FAIL baseline for spec requirement.

Run: DevTools Network Request Blocking `*web3forms*`, submit valid.
Expected: FAIL — error shown, should succeed via Supabase.

- [ ] **Step 2: Run check to verify it fails**

Run: same block test.
Expected: FAIL — confirms sequential logic.

- [ ] **Step 3: Write minimal implementation**

Replace body after bot/rate guards with:

```js
const sendBtn = document.getElementById("cfSend");
const okEl = document.getElementById("cfOk");
const errEl = document.getElementById("cfErr");
function okNote(msg) { if (okEl) { okEl.textContent = msg; okEl.hidden = false; } if (errEl) { errEl.textContent = ""; errEl.hidden = true; } }
function errNote(msg) { if (errEl) { errEl.textContent = msg; errEl.hidden = false; } }
sendBtn.disabled = true;
sendBtn.textContent = "Sending…";
function withTimeout(promise, ms) {
  let t = 0;
  const gate = new Promise(function (_, rej) { t = setTimeout(function () { rej(new Error("timeout")); }, ms); });
  return Promise.race([promise.then(function (v) { clearTimeout(t); return v; }, function (e) { clearTimeout(t); throw e; }), gate]);
}
const n = document.getElementById("cfName").value.trim();
const m = document.getElementById("cfEmail").value.trim();
const t = document.getElementById("cfMsg").value.trim();
const web3 = withTimeout(fetch("https://api.web3forms.com/submit", {
  method: "POST",
  headers: { "Content-Type": "application/json", "Accept": "application/json" },
  body: JSON.stringify({ access_key: "1d503400-36a0-4ad8-9b26-ca8d14ed20fe", name: n, email: m, message: t, subject: "New portfolio message from " + n, from_name: n })
}).then(function (r) { return r.json().catch(function () { return {}; }).then(function (d) { return r.ok && !!d.success; }); }).catch(function () { return false; }), 10000);
const sb = (window.supabase ? withTimeout((function () {
  try {
    const db = supabase.createClient("https://sbqxizjtgdfixvnwtbry.supabase.co", "sb_publishable_jky21GxgSF35_XYQWRnnbw_na-NBhOW");
    return db.from("messages").insert({ name: n, email: m, message: t }).then(function (res) { return !res.error; });
  } catch (_) { return Promise.resolve(false); }
})(), 10000).catch(function () { return false; }) : Promise.resolve(false));
Promise.allSettled([web3, sb]).then(function (rs) {
  const ok = rs.some(function (r) { return r.status === "fulfilled" && r.value === true; });
  if (ok) {
    okNote("Message sent! I'll get back to you within a day.");
    cf.reset();
    cf.dataset.startedAt = String(Date.now());
    try { localStorage.setItem(CF_LS, String(Date.now())); } catch (_) {}
  } else {
    errNote("Couldn't send just now — email me directly at j.doruca109@gmail.com.");
  }
}).finally(function () {
  sendBtn.disabled = false;
  sendBtn.textContent = "Send message";
});
```

Keep `return` after wiring (handler is sync, promises async). Do not add `alert`, do not `mailto:` auto-open, do not log RLS text to UI.

- [ ] **Step 4: Run check to verify it passes**

Run three cases with valid data (wait 60s between or clear `localStorage.removeItem('v2-last-send')` + reload for fresh `startedAt`):
1. Block `*web3forms*` → submit → expect `#cfOk` visible (Supabase path).
2. Block `*supabase*` → submit → expect `#cfOk` visible (email path).
3. Block both → submit → expect `#cfErr` text `Couldn't send…`, fields preserved, button re-enabled.
Expected: PASS all three.

- [ ] **Step 5: Commit**

```bash
git add Final_portfolio/app-2.js Final_portfolio/index.html
git commit -m "feat(contact): parallel dual-write with timeout"
```

---

### Task 5: Regression + cleanup verification (basic only)

**Files:**
- Modify: none (verification only)
- Test: manual checklist across `index.html`, `admin.html`, `login.html`

**Interfaces:**
- Consumes: Tasks 1-4 output
- Produces: verified working tree, no console errors, inbox intact.

- [ ] **Step 1: Write the failing check (define expected clean state)**

Checklist file (run in head, no code): console has 0 errors on load (ignore GH chart 404 offline), theme toggle works, mobile burger opens, GSAP reveals run, chat widget opens, `login.html` → `admin.html` with owner email loads inbox, contact success row appears in inbox top.

Run: `python -m http.server 8000` in `Final_portfolio/`, open `http://localhost:8000/index.html`, DevTools Console.
Expected: FAIL if any red error from our edits (e.g., `cfCompany is null`, `CF_LS is not defined`).

- [ ] **Step 2: Run check to verify current status**

Run: reload, click theme, burger, chat Fab, submit one valid contact (clear rate-limit first), login to admin, confirm new message.
Expected: note any FAIL item to fix inline in Tasks 1-4 (no new code here).

- [ ] **Step 3: Minimal fix pass (if needed, else no-op)**

If Step 2 found e.g. `aria-invalid` border missing in light theme, confirm Task 2 CSS present. If duplicate `copyEmail` handler only binds first button (pre-existing), leave as-is per spec out-of-scope. No new code unless Step 2 FAILs.

- [ ] **Step 4: Run final verification**

Run: repeat Step 2 end-to-end on Chrome + Firefox, 360px width, keyboard-only Tab → Enter submit. Confirm:
```js
document.querySelectorAll('[aria-invalid="true"]').length===0 && document.getElementById('cfOk').hidden===false
```
after valid submit. `git status --short` shows only `M Final_portfolio/index.html`, `M Final_portfolio/styles-2.css`, `M Final_portfolio/app-2.js`, plus new docs files. No `recovery-codes.txt`, no `.env`, no inbox dump.

Expected: PASS.

- [ ] **Step 5: Commit docs (code already committed per-task)**

```bash
git add docs/superpowers/specs/2026-10-04-contact-polish-design.md docs/superpowers/plans/2026-10-04-contact-polish.md
git commit -m "docs(contact): add spec and plan for polish"
```

---
