# Client-side security review: localhost:8765

`http://localhost:8765/login/?next=/dashboard`  ·  2026-10-19T07:35:59.914Z  ·  review.js 3.0  ·  grade **F**

| High | Medium | Low | Info |
|---|---|---|---|
| 2 | 8 | 3 | 14 |

Passive review from the browser console; nothing was exploited. Items marked *(manual)* need confirmation.

## High (2)

### CKY-01 · Session-looking cookie readable by JavaScript (no HttpOnly)
- **Evidence:** sessionid
- **Advice:** Set `HttpOnly` on session cookies; any XSS can otherwise read document.cookie and hijack the session. Verify in Application/Storage → Cookies.
- **Ref:** WSTG-SESS-02 / CWE-1004

### TRN-01 · Page served over plain HTTP
- **Evidence:** http://localhost:8765/login/?next=/dashboard
- **Advice:** Serve over HTTPS and redirect http:// → https:// (301). Credentials and cookies are readable on the wire.
- **Ref:** WSTG-CRYP-03 / CWE-319

## Medium (8)

### CSP-01 · No Content-Security-Policy
- **Evidence:** header and <meta> absent
- **Advice:** Deploy a nonce-based policy: `script-src 'nonce-…' 'strict-dynamic'; object-src 'none'; base-uri 'self'`. It turns XSS from critical into contained.
- **Ref:** WSTG-CONF-12 / CWE-1021

### FRM-04 · No CSRF token field in password form *(manual)*
- **Evidence:** login · fields: email,password,remember
- **Advice:** Login CSRF lets an attacker log the victim into the attacker's account. Use a synchronizer token, or SameSite=Lax/Strict cookies + Origin check. (A token may travel in a header via JS — verify in Network.)
- **Ref:** WSTG-SESS-05 / CWE-352

### HDR-03 · Page can be framed by any site (clickjacking)
- **Evidence:** no X-Frame-Options, no CSP frame-ancestors
- **Advice:** Send `Content-Security-Policy: frame-ancestors 'none'` (or 'self') plus `X-Frame-Options: DENY` for legacy browsers. Verify by loading the URL in an <iframe> on another origin.
- **Ref:** WSTG-CLNT-09 / CWE-1021

### JS-02 · Script writes auth tokens to Web Storage
- **Evidence:** /static/app.js · localStorage.setItem('authToken'
- **Advice:** See STO-01: prefer HttpOnly cookies or in-memory tokens.
- **Ref:** WSTG-CLNT-12 / CWE-922

### JS-03 · DOM-XSS sinks with untrusted sources in /static/app.js *(manual)*
- **Evidence:** sinks: innerHTML · sources: location.search, URLSearchParams, localStorage
- **Advice:** Static co-occurrence only. The taint run (TNT-01) catches flows that read sources after load; code that reads location/window.name only at startup needs a reload with the canary in place (DOM Invader does this automatically), e.g. open the page with #rvtest=<img src=x> and watch Elements. Confirm: anything from URL/postMessage/referrer/storage/API must go through textContent, DOMPurify or Trusted Types before a sink.
- **Ref:** WSTG-CLNT-01 / CWE-79

### RED-01 · Redirect-style parameter in the URL *(manual)*
- **Evidence:** next=/dashboard
- **Advice:** Open-redirect candidate. Try: https://example.org · //example.org · /\example.org · https:example.org · %2F%2Fexample.org · https://localhost:8765.example.org · https://example.org%23@localhost:8765. After login only relative paths or an allow-list may be followed.
- **Ref:** WSTG-CLNT-04 / CWE-601

### SCR-01 · Third-party scripts on an authentication page *(manual)*
- **Evidence:** cdn.analytics.example
- **Advice:** Every script can read the DOM (password fields included) and exfiltrate. Minimise on auth/payment pages; pin with SRI; consider a separate origin.
- **Ref:** CWE-829

### STO-01 · JWT stored in localStorage
- **Evidence:** authToken
- **Advice:** Web Storage is readable by every script on the origin. Prefer an HttpOnly cookie; if a bearer token is unavoidable keep it in memory and short-lived.
- **Ref:** WSTG-CLNT-12 / CWE-922

## Low (3)

### FRM-05 · Password field autocomplete=off
- **Evidence:** login.password
- **Advice:** Use `current-password` / `new-password`; blocking password managers lowers real security.
- **Ref:** WSTG-ATHN-01

### HDR-08 · Authentication page is cacheable
- **Evidence:** Cache-Control: (absent)
- **Advice:** Send `Cache-Control: no-store` on login, reset and authenticated pages.
- **Ref:** WSTG-ATHN-06 / CWE-525

### SCR-02 · Cross-origin scripts without Subresource Integrity
- **Evidence:** cdn.analytics.example/tag.js
- **Advice:** Add `integrity="sha384-…" crossorigin="anonymous"`.
- **Ref:** CWE-353

## Info (14)

### HDR-05 · No X-Content-Type-Options: nosniff
- **Evidence:** header absent
- **Advice:** Add it to stop MIME sniffing of uploads/scripts.
- **Ref:** WSTG-CONF-12

### HDR-06 · No Referrer-Policy
- **Evidence:** header/meta absent
- **Advice:** Set `Referrer-Policy: strict-origin-when-cross-origin` or stricter.
- **Ref:** WSTG-INFO-08 / CWE-200

### HDR-09 · Version/infrastructure disclosure in server
- **Evidence:** server: SimpleHTTP/0.6 Python/3.13.16, nginx/1.18.0
- **Advice:** Strip versions and internal names from response headers.
- **Ref:** WSTG-INFO-02 / CWE-200

### HDR-09 · Version/infrastructure disclosure in x-powered-by
- **Evidence:** x-powered-by: PHP/8.1.2
- **Advice:** Strip versions and internal names from response headers.
- **Ref:** WSTG-INFO-02 / CWE-200

### HDR-12 · No Permissions-Policy
- **Evidence:** header absent
- **Advice:** Restrict powerful features: `Permissions-Policy: camera=(), microphone=(), geolocation=()`.
- **Ref:** WSTG-CONF-12

### HDR-13 · No Cross-Origin-Opener-Policy on auth page
- **Evidence:** header absent
- **Advice:** `Cross-Origin-Opener-Policy: same-origin` severs window references from cross-origin openers (tabnabbing, XS-Leaks).
- **Ref:** CWE-1021

### HDR-14 · Content-Type without charset
- **Evidence:** text/html
- **Advice:** Declare charset=utf-8.
- **Ref:** CWE-116

### INV-01 · API endpoints referenced in client code (4) *(manual)*
- **Evidence:** GET /api/search · POST /session · POST /api/cart · ? /api
- **Advice:** Full list with methods/params in __review.endpoints; feed into Burp/ZAP scope.
- **Ref:** WSTG-INFO

### INV-02 · Checks this tool cannot do — test manually *(manual)*
- **Evidence:** auth logic, server-side validation, authorisation
- **Advice:** User enumeration (error text/status/timing for valid vs unknown user) · brute-force / lockout / CAPTCHA · session id rotation on login, invalidation on logout, concurrent sessions · MFA ordering (reach protected endpoints after step 1) · reset-link single-use + expiry · real XSS confirmation on JS-03/TNT-01 · open-redirect confirmation on RED-* · IDOR on JS-21 / INV-01 · CORS with a foreign Origin header · clickjacking via a local iframe · CSWSH on JS-07.
- **Ref:** WSTG-ATHN / WSTG-SESS / WSTG-ATHZ

### JS-00 · 1 cross-origin script(s) not analysed (CORS) *(manual)*
- **Evidence:** cdn.analytics.example
- **Advice:** Fetch them outside the browser (curl) and grep for the same patterns.


### JS-15 · Trusted Types not enforced
- **Evidence:** no require-trusted-types-for directive
- **Advice:** `require-trusted-types-for 'script'` makes every DOM sink refuse raw strings — the strongest DOM-XSS mitigation available.
- **Ref:** CWE-79

### RED-04 · Links on this page carry redirect parameters *(manual)*
- **Evidence:** /login
- **Advice:** Each is an open-redirect candidate (RED-01 payloads).
- **Ref:** WSTG-CLNT-04

### STO-08 · JWT uses HMAC (HS256) *(manual)*
- **Evidence:** localStorage:authToken
- **Advice:** Fine with a strong secret; weak secrets are brute-forceable offline (jwt_tool / hashcat on your own token).
- **Ref:** CWE-347

### STO-11 · JWT carries authorisation claims *(manual)*
- **Evidence:** localStorage:authToken · "role":"user"
- **Advice:** Only base64, not encrypted. Verify the server re-checks authorisation and does not trust a client-modified copy.
- **Ref:** CWE-312

## Endpoints referenced in client code

| Method | URL | Params | From |
|---|---|---|---|
| GET | `/api/search` | q | /static/app.js |
| POST | `/session` |  | /static/app.js |
| POST | `/api/cart` |  | /static/app.js |
| ? | `/api` |  | /static/app.js |

## Response headers

```
content-length: 874
content-type: text/html
date: Mon, 19 Oct 2026 07:35:58 GMT
last-modified: Sun, 18 Oct 2026 20:09:14 GMT
server: SimpleHTTP/0.6 Python/3.13.16, nginx/1.18.0
x-powered-by: PHP/8.1.2
```
