# review.js

**A passive client-side security review you run from the browser's Console.**
Paste one file into DevTools on a page you are allowed to test, press Enter, and in about a second
you get a graded list of findings — each with the evidence, what to do about it, and the OWASP /
CWE reference — plus a Markdown report you can paste straight into your write-up.

It never attacks anything. It reads what the browser already has: the page, its headers, cookies,
storage and scripts. That makes it safe to run on every page of an engagement, inside your
logged-in session, as the first thing you do before reaching for Burp or ZAP.

> **Authorisation first.** Only run this on systems you own or have written permission to test.
> The script itself is passive, but the manual follow-ups it suggests (redirect payloads, CORS
> probes) are not.

---

## What is in this folder

| File | What it is | When you use it |
|---|---|---|
| **`review.js`** | The tool. One self-contained JavaScript file, no dependencies, Chrome + Firefox. | Paste it into the Console. This is the only file you actually need. |
| `review-bookmarklet.txt` | The same tool, minified and prefixed with `javascript:` so it fits in a bookmark. | Make a bookmark with this as the URL; click it on any page instead of pasting. |
| `review.user.js` | The same tool wrapped as a Tampermonkey / Violentmonkey userscript. | Install once; press `Ctrl+Shift+R` on any page, or pick "Run review.js" from the extension menu. |
| `sample-report.md` | A real Markdown report produced by `__review.md()` on a deliberately broken test site. | Read it to see what output looks like before you run anything. |
| `README.md` | This file. | — |
| `LICENSE` | MIT. | Use it, fork it, ship it; keep the notice. |

The three "tool" files are generated from `review.js` and run the same checks; pick whichever is
convenient.

---

## Your first ten minutes

You need: a browser (Chrome, Edge, Brave or Firefox), a page you are allowed to test, and
`review.js` open in a text editor.

**1. Open the page and log in** with your test account. The review is most useful on the pages that
matter: login, account settings, checkout, anything with a form or a token.

**2. Open the Console.** Press `F12`, then click the **Console** tab. (Mac: `Cmd+Option+J`.)
If the Console shows a warning about pasting — Chrome does this the first time — type
`allow pasting` and press Enter, as it asks.

**3. Paste the whole of `review.js` and press Enter.** You will see something like:

```
review.js 1.0.0  ·  shop.example.com/login  ·  grade F
┌───────────┬───────┐
│ high      │ 3     │
│ medium    │ 8     │
│ low       │ 3     │
│ info      │ 14    │
│ auth page │ true  │
│ scripts   │ 4     │
│ endpoints │ 7     │
└───────────┴───────┘
▸ HIGH 3
▸ MEDIUM 8
▸ LOW 3
▸ INFO 14
```

**4. Click the `▸ HIGH` group open, then click a finding.** Every finding has the same four lines:

```
▸ CKY-01 Session-looking cookie readable by JavaScript (no HttpOnly)
    evidence: sessionid
    advice:   Set `HttpOnly` on session cookies; any XSS can otherwise read document.cookie
              and hijack the session. Verify in Application/Storage → Cookies.
    ref:      WSTG-SESS-02 / CWE-1004
```

- **ID** (`CKY-01`) — stable name for the check; use it in your notes and when searching this README.
- **evidence** — copied from the page, so you can show exactly what was observed.
- **advice** — what the developer should change, and how you can confirm it by hand.
- **ref** — the OWASP Web Security Testing Guide section and/or CWE number to cite in the report.
- **(verify by hand)** after the title means the tool found a *pointer*, not a verdict. A third-party
  script on the login page might be the customer's SSO provider; a `innerHTML` might only ever
  receive constants. Go and look before you write it up.

**How the next steps work.** Running `review.js` once leaves an object called `__review` behind on
the page. Steps 5–7 are commands you type on the empty `>` line at the bottom of the same Console,
one at a time, after the findings have printed. You do not remove or re-paste the script; it stays
loaded until you reload the page or navigate to another URL (then paste it again).

**5. Get the report.** Type in the Console:

```javascript
__review.copy('md')
```

The Markdown report is now on your clipboard. Paste it into your notes. It contains the summary
table, every finding, the list of API endpoints the JavaScript talks to, the libraries detected, and
the response headers.

**6. Use the app while the monitors watch.** Type:

```javascript
__review.monitor()
```

Then do things on the page: log in, search, change a setting. Every `fetch`/XHR request and
response, every WebSocket message, every `postMessage`, every write to `innerHTML`, `eval`,
`localStorage` and `document.cookie` is now logged with the line of code that did it. This is how
you learn what the application actually does with your data. The yellow `[monitor]` lines appear on
their own while you click; collapse them with the triangle or type `-monitor` in the Console filter box
to hide them for a moment. Reload the page to switch the monitors off.

**7. Go deeper on DOM XSS.** Type:

```javascript
__review.taint({storage: true, cookie: true})
```

The tool now plants unique markers in `location.hash`, `window.name`, a `postMessage`, every
`localStorage` value and a cookie (all restored afterwards), and watches 25 dangerous sinks. If a
marker arrives in one, you get a `TNT-01` finding with the value and the call stack: a confirmed
source→sink flow, not a guess.

That is the whole workflow. Repeat 3–7 on each interesting page.

---

## Reading the grade

| Grade | Meaning |
|---|---|
| **A** | nothing above *info* |
| **B** | only *low* findings |
| **C** | one *medium* |
| **D** | several *medium* |
| **F** | at least one *high* |

The grade is a triage aid for you, not a score for the customer. A page can be an **A** here and
still have a broken authorisation model, because no client-side check can see the server.

Rough severity guide: **high** = direct credential/session/code-execution exposure (secret in the
bundle, session cookie without HttpOnly, confirmed taint to `innerHTML`, static CSP nonce);
**medium** = a real weakness that needs a second condition to exploit (token in localStorage,
missing CSRF token, vulnerable library version); **low** = hardening gaps; **info** = inventory and
things to be aware of.

---

## What the tool cannot tell you (and what to do instead)

The last *info* finding on every run, `INV-02`, lists this. In short, anything that happens on the
server is invisible from the browser:

| Question | How you answer it |
|---|---|
| Does the error differ for a wrong password vs an unknown user? (user enumeration) | Two logins in Network tab; compare status, text and timing. |
| Is there rate limiting / lockout? | A handful of failed logins; watch for 429, delays, CAPTCHA. |
| Does the session id change on login, and die on logout? | Note the cookie before/after login; reuse an old token after logout (Burp Repeater). |
| Can MFA be skipped? | After password step, hit a protected endpoint from `INV-01` directly. |
| Is the `JS-03` sink really exploitable? | `__review.taint()` first; then a real payload on a test instance. |
| Is the `RED-01` redirect really open? | Try the payloads listed in the finding, with your own account. |
| Can I see another user's object? (IDOR) | Take ids from `JS-21` / `INV-01`, swap them, same session. |

Use the endpoint table in the report as your scope for Burp / ZAP.

---

## Configuration (optional)

Set this **before** pasting the script to change defaults:

```javascript
window.__REVIEW_CFG = {
  fetchScripts: true,   // download and scan same-origin scripts (set false on huge SPAs if slow)
  sourcemaps:   true,   // HEAD-probe for public .map files
  taint:        true,   // run the light taint pass automatically (hash, window.name, postMessage)
  gadgets:      true,   // test jQuery/lodash/... for prototype-pollution gadgets (cleaned up)
  maxScriptKB:  1500,   // per-script size cap
  quiet:        false   // true = no console output, just window.__review
}
```

After a run, everything lives on `window.__review`:

```
__review.findings     array of {id, severity, area, title, evidence, advice, ref, manual}
__review.md()         Markdown report      __review.copy('md')
__review.json()       JSON report          __review.copy('json')
__review.taint(opts)  deeper taint run     __review.monitor()   runtime monitors
__review.endpoints    [{url, methods, params, from}]      __review.libraries   [{name, version, via}]
__review.headers      __review.csp      __review.globals     __review.taintHits
__review.help()
```

---

## Running it in Firefox

Firefox has no "Snippets" panel; use the Console's multi-line editor: press `Ctrl+B` in the
Console, paste (or `Ctrl+O` to open `review.js` from disk), then `Ctrl+Enter`. The editor remembers
its contents. Everything else works the same; the one difference is that cookie attributes
(`Secure`/`SameSite`) are read from the **Storage** panel rather than detected automatically, and
the tool says so (`CKY-06`).

---

## Reference: every check

Each finding carries an ID (`AREA-nn`), a severity (high / medium / low / info), evidence
copied from the page, concrete remediation advice, and a reference to the OWASP Web Security
Testing Guide section and/or CWE. Findings marked **manual** are pointers that need a human
to confirm (e.g. "this third-party script may be your SSO provider").

| Area | IDs | Checks |
|---|---|---|
| Transport | `TRN` | plain HTTP, HSTS presence / max-age / includeSubDomains, active mixed content, http:// links and form actions |
| Headers | `HDR` | framing (XFO / frame-ancestors, obsolete ALLOW-FROM) · nosniff · Referrer-Policy · Cache-Control on auth pages · version/infrastructure disclosure · CORS credentials with foreign origin · Permissions-Policy · COOP · charset · obsolete X-XSS-Protection |
| CSP | `CSP` | missing / report-only / meta-only · no script-src or default-src · `unsafe-inline` without nonce/hash · `unsafe-eval` · **nonce too short, non-base64, or identical across two page loads** · scheme/`*` sources · wildcard hosts · **~30 known bypass hosts** (JSONP, AngularJS, user-content CDNs, PaaS wildcards) · `'self'` without nonces · nonces without `strict-dynamic` / `strict-dynamic` without nonces · IP/localhost/http: sources · `object-src`, `base-uri`, `form-action` · permissive default-src · deprecated directives · no reporting · `connect-src *` · Trusted Types |
| Cookies | `CKY` | session-looking cookies readable by JS (no HttpOnly), JWTs in JS-readable cookies, Secure / SameSite via Cookie Store API where available, `__Host-` prefix, parent-domain scope, long-lived sessions |
| Storage | `STO` | tokens/JWTs in Web Storage · JWT inspection: alg=none, HMAC, `jku`/`x5u` headers, missing/long/expired `exp`, authorisation claims, **service-role / admin tokens** · PII, IBAN/card numbers, password-named keys, IndexedDB presence |
| Forms | `FRM` | password forms: GET, http://, cross-origin action, missing CSRF token, autocomplete=off, pre-filled, type=text secrets · hidden fields the server might trust (role/price/redirect) · state-changing forms without CSRF · file uploads · sensitive inputs without autocomplete hints · password fields outside forms |
| Redirects / URL | `RED` | redirect parameters (with the exact bypass payloads to try), secrets in query string, tokens in fragment (implicit OAuth), links carrying redirect params, foreign `<base href>` |
| Scripts | `SCR` | third-party scripts (escalated on auth pages), missing SRI on cross-origin scripts and stylesheets, inline scripts without nonces, inline `on*=` handlers, `javascript:`/`data:` URLs, tag managers and session-replay on auth pages, import maps |
| DOM | `DOM` | cross-origin iframes without sandbox, http iframes, `_blank` without noopener, HTML comments with TODO/password/internal IPs, hidden privileged-looking UI, permissive meta referrer, meta refresh, email harvesting, unmasked password-like inputs |
| Libraries | `LIB` | **~50 libraries** detected via globals *and* script URLs, with CVE-annotated vulnerable ranges: jQuery (+UI/Mobile/Validation), AngularJS, Angular, React/ReactDOM, Next.js (incl. CVE-2025-29927), Vue, lodash, Underscore, moment, Bootstrap, bootstrap-select, Handlebars, Mustache, DOMPurify, Axios, Dojo, Prototype, Ember, Knockout, CKEditor 4/5, TinyMCE, Quill, Froala, Summernote, marked, markdown-it, highlight.js, Prism, DataTables, Chart.js, PDF.js (CVE-2024-4367), video.js, JSZip, SheetJS, jsPDF, CryptoJS, jsrsasign, ua-parser-js (incl. the compromised releases), Immer, js-yaml, Swagger UI, Socket.IO, htmx, Alpine, Select2 · dev builds / devtools enabled · secrets or PII in `__NEXT_DATA__` / `__INITIAL_STATE__` / `__NUXT__` / Apollo / Remix / SvelteKit state |
| Globals | `GLB` | app-added globals that look like secrets (diffed against a clean iframe) · Object.prototype already polluted · relaxed `document.domain` · **gadget tests**: feeds `{"__proto__":{…}}` through jQuery.extend(true), lodash.merge/defaultsDeep/set, angular.merge, dojo.setObject, Handlebars.Utils.extend, Chart.helpers.merge and reports which ones actually pollute (cleaned up immediately) |
| JavaScript source | `JS` | over inline scripts, `on*=` attributes and every same-origin script (fetched, up to 40 files × 1.5 MB): **~35 secret detectors** (AWS, Stripe live/test/restricted, GitHub, Slack tokens + webhooks, Discord, Google API/OAuth, Firebase config, OpenAI, Anthropic, HuggingFace, SendGrid, Mailgun, Twilio, Mapbox, npm, PyPI, Azure keys/SAS, Heroku, private keys, basic-auth URLs, Bearer literals, JWTs — decoded, privileged claims escalated — and generic `api_key = "…"`), tokens written to storage, 17 DOM-XSS **sinks** paired with 10 **sources**, storage→sink flows, postMessage listeners without origin check, `postMessage(…,'*')`, ws://, CSWSH reminder, document.domain, secrets in console.log, client-side hashing, Math.random near secrets, JSONP, base64-as-encryption, deep-merge code touching `__proto__`, sanitizer presence, debugger, source-map references and **publicly served source maps**, Trusted Types |
| Taint | `TNT` | **Canary taint tracking**: unique markers are placed in `location.hash`, `window.name` and sent via `postMessage` (and, with `__review.taint({storage:true,cookie:true})`, appended to every storage value and a cookie, all restored afterwards). 25 sinks are hooked (innerHTML/outerHTML/insertAdjacentHTML, document.write, eval/Function/setTimeout, script.src/text, iframe.src/srcdoc, a.href, form.action, setAttribute, DOMParser, createContextualFragment, location.assign/replace, window.open, fetch/XHR URLs, storage.setItem, document.cookie, jQuery html/append/… and `$(html)`). A finding is raised **only when a marker actually reaches a sink**, with the value and call stack — no false positives from co-occurrence. Sources read only at page load need a reload with the marker in the URL (noted in the advice). |
| Endpoints | `INV`, `JS-16/21` | fetch/axios/jQuery/XHR/WebSocket calls with **HTTP method and query parameters**, privileged-looking paths (admin/internal/export…), object-id paths (IDOR candidates); full table in `__review.endpoints` and the Markdown report |
| Workers / permissions | `SW`, `PERM` | registered service workers and scopes, permissions already granted, cross-origin isolation |
| Inventory | `INV` | every API endpoint referenced in client code (seed for Burp/ZAP scope), and the explicit list of what must be tested by hand |

## Credits / prior art

The ideas are borrowed, with thanks, from tools that do each part more deeply: PortSwigger **DOM Invader**
(canary taint tracking, prototype-pollution gadgets), **Retire.js** (library/CVE matching), Google
**CSP Evaluator** (bypass hosts, nonce rules), Bishop Fox **jsluice** (endpoint extraction) and
**TruffleHog** (secret detectors). Use those when you need the full depth; use this when you need all of
it in one paste, inside the logged-in session, with a report at the end.



## What it deliberately does not do

- No injection, no brute force, no requests to other origins, no modification of the page.
- No server-side judgement: user enumeration, rate limiting, session rotation, MFA ordering,
  IDOR and real XSS confirmation are listed under `INV-02` as manual work (Burp/ZAP Repeater,
  a second browser profile).
- Cross-origin scripts are not read (CORS). Fetch them outside the browser and grep for the
  same patterns, or run Retire.js on them.
- The taint run only sees sinks reached *after* injection; code that reads `location.hash` or
  `window.name` once at startup is caught by the static `JS-03` check and needs a reload with
  `#rvtest=<img src=x>` to confirm.
- The library table is hand-maintained; Retire.js has far more entries. Anything in `LIB-00`
  not covered should be checked there.
- Cookie `HttpOnly` is inferred (readable ⇒ missing); `Secure`/`SameSite` need the Cookie
  Store API (Chrome) or the Storage panel (Firefox).


## Ethics

Run only on systems you own or have written authorisation to test. The script is passive, but
the manual follow-ups it recommends (redirect payloads, CORS probes, CSWSH) are not, and are
illegal without permission in most jurisdictions.

## Changelog

- **1.0.1** — Taint tracking no longer reports the tool's own internal requests (e.g. `location.hash → fetch(url)` with a stack entirely inside "debugger eval code" / `VM…`). Hits whose call stack contains no frame from the site's own scripts are dropped; the advice text now says how to recognise a self-hit.
- **1.0.0** — First public release.
