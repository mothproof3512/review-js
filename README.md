# review.js — passive client-side security review from the browser console

One file, no dependencies, Chrome + Firefox. Version 3 adds **taint tracking with canaries**
(the DOM Invader idea), a **library CVE database** (Retire.js style, ~50 libraries), a **CSP
evaluator** modelled on Google's (bypass hosts, nonce length and reuse, strict-dynamic logic),
**prototype-pollution gadget tests**, **endpoint extraction with HTTP methods** (jsluice style)
and ~35 **secret detectors** including decoding of embedded JWTs (TruffleHog style). Paste it into the Console on any page you are
authorised to test and you get a graded, referenced list of client-side findings in under a
second, plus Markdown/JSON export for your report. Nothing is exploited: every check reads the
page, its headers, cookies, storage and same-origin scripts; the only requests it makes are a
re-fetch of the current page (to read headers), the page's own same-origin scripts, and optional
`HEAD` probes for `.map` files.

```
__review.findings          every finding: {id, severity, area, title, evidence, advice, ref, manual}
__review.md()              Markdown report              __review.copy('md')
__review.json()            JSON report                  __review.copy('json')
__review.taint({storage:true, cookie:true})   deeper taint run: canaries also into storage/cookie (auto-restored)
__review.monitor()         opt-in runtime monitors (fetch, XHR, WebSocket, postMessage,
                           innerHTML, eval, document.write, storage, cookie, message listeners)
__review.endpoints / .libraries / .headers / .csp / .globals / .taintHits
__review.help()
```

Configure before running (optional):

```javascript
window.__REVIEW_CFG = { fetchScripts: true, sourcemaps: true, taint: true, gadgets: true, maxScriptKB: 1500, quiet: false }
```

## Three ways to run it

| How | Chrome | Firefox |
|---|---|---|
| **Paste** `review.js` into the Console | F12 → Console → paste → Enter | same |
| **Saved snippet** | Sources → Snippets → New → paste → `Ctrl+Enter` | Console → `Ctrl+B` (multi-line editor) → `Ctrl+O` open `review.js` → `Ctrl+Enter` |
| **Bookmarklet** | new bookmark, URL = contents of `review-bookmarklet.txt` | same |
| **Userscript** | Tampermonkey/Violentmonkey → install `review.user.js` → `Ctrl+Shift+R` on any page, or the extension menu | same |

The bookmarklet and userscript are generated from `review.js`; they are byte-for-byte the same checks.

## What it checks

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

## Grade

`A` nothing above info · `B` lows only · `C` one medium · `D` several mediums · `F` any high.
It is a triage aid, not a certification: a page can be `A` here and still have a broken
authorisation model, which no client-side check can see.

## Monitors (`__review.monitor()`)

Wraps the APIs an attacker would use and logs every call with a stack trace, collapsed in the
console: `fetch`/XHR with request body and response preview, WebSocket open/send/receive,
outgoing `postMessage` and incoming `message` events with origin, `innerHTML` writes that
contain markup, `eval`, `document.write`, `localStorage/sessionStorage.setItem`,
`document.cookie =`, and new `message` listeners. Turn it on, then use the application:
log in, search, change settings. Everything the JavaScript does with your data is now visible,
with the line of code that did it. Reload to disable.

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

## Interpreting a run

1. Fix or verify everything **high** first; each one is a direct credential/session exposure.
2. Any **`TNT-01`** is a confirmed source→sink flow: reproduce it with a real payload on a test
   instance. `JS-03` (static co-occurrence) is the list of places to look next.
3. `RED-01` lists the exact payloads to try for open redirect; do it with your own account.
4. `INV-01` endpoints plus `JS-16` privileged endpoints are your authorisation test list.
5. Paste `__review.copy('md')` into the report; the evidence lines are already quoted.

## Ethics

Run only on systems you own or have written authorisation to test. The script is passive, but
the manual follow-ups it recommends (redirect payloads, CORS probes, CSWSH) are not, and are
illegal without permission in most jurisdictions.
