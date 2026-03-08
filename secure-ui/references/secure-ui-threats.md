# Secure UI Threat Catalog

Reference for the `secure-ui` skill. Each entry covers: what the attack is, how it
manifests in UI, and how to mitigate it at the UI layer.

Sources: Apple Secure Coding Guide, OWASP Cheat Sheet Series, OWASP Top 10

---

## Table of Contents

1. [Cross-Site Scripting (XSS)](#1-cross-site-scripting-xss)
2. [Cross-Site Request Forgery (CSRF)](#2-cross-site-request-forgery-csrf)
3. [Clickjacking](#3-clickjacking)
4. [Open Redirect](#4-open-redirect)
5. [Social Engineering / Phishing](#5-social-engineering--phishing)
6. [Insecure Data Storage (Client-Side)](#6-insecure-data-storage-client-side)
7. [Sensitive Data Exposure in UI](#7-sensitive-data-exposure-in-ui)
8. [Tab-Napping (Opener Hijacking)](#8-tab-napping-opener-hijacking)
9. [Insecure Authentication UX](#9-insecure-authentication-ux)
10. [Supply Chain / Third-Party Script Injection](#10-supply-chain--third-party-script-injection)
11. [Insecure File Handling UI](#11-insecure-file-handling-ui)
12. [Permission / Authorization UX Failures](#12-permission--authorization-ux-failures)
13. [Homograph / IDN Spoofing](#13-homograph--idn-spoofing)

---

## 1. Cross-Site Scripting (XSS)

**What it is:** Attacker injects malicious scripts into a page; victims' browsers execute
them in the context of the trusted site. Can steal cookies, session tokens, keystrokes,
or perform actions on the user's behalf.

**Types:**
- **Stored XSS** — malicious script saved in DB, served to all users who load that content
- **Reflected XSS** — script in URL parameter, reflected back in response
- **DOM-based XSS** — script injected via client-side DOM manipulation (e.g. `innerHTML`)

**UI attack surfaces:**
- Comment fields, user profiles, chat messages rendered as HTML
- URL parameters inserted into the DOM
- `dangerouslySetInnerHTML` / `v-html` / `[innerHTML]` with untrusted data
- `href="javascript:..."` or `onclick` attributes built from user input

**Mitigations:**
- Use framework-native rendering (React JSX, Vue templates) — auto-escape by default
- Sanitize with DOMPurify when rich HTML is genuinely needed
- Set a Content Security Policy (CSP) header
- Never use `eval()`, `Function()`, `setTimeout(string)` with user data
- Use `element.textContent` not `element.innerHTML` for inserting user text

---

## 2. Cross-Site Request Forgery (CSRF)

**What it is:** Attacker tricks an authenticated user's browser into making an unwanted
state-changing request to your app. Browser auto-sends session cookies, so server sees
it as legitimate.

**UI attack surfaces:**
- Any state-changing action (POST form, delete button, settings change)
- AJAX requests that rely solely on cookies for authentication

**Mitigations:**
- CSRF token on all state-changing requests (unique per session, server-generated, not in a cookie or URL)
- `SameSite=Strict` or `SameSite=Lax` cookies (modern defense)
- Use framework-native CSRF protection (don't hand-roll)
- Validate `Origin` and `Referer` headers server-side
- Never use GET for state-changing operations

---

## 3. Clickjacking

**What it is:** Attacker embeds your site in an invisible iframe overlaid on their
malicious page; user thinks they're clicking the attacker's UI but actually clicks yours.

**UI attack surfaces:**
- Any page or component that can be framed
- Particularly dangerous for: payment confirmations, permission grants, delete actions

**Mitigations:**
- `X-Frame-Options: DENY` response header
- `Content-Security-Policy: frame-ancestors 'none'` (modern equivalent)
- For legitimate embedded contexts: `frame-ancestors 'self' https://trusted-parent.com`
- Frame-busting JavaScript (weaker, use headers as primary defense)

---

## 4. Open Redirect

**What it is:** App redirects to a user-supplied URL without validation; attacker uses
your trusted domain as a launchpad for phishing (e.g. `yourapp.com/redirect?to=evil.com`).

**UI attack surfaces:**
- Login redirect (`?next=`, `?return_to=`, `?redirect_uri=`)
- Post-action redirects
- "Continue to..." links

**Mitigations:**
- Allowlist valid redirect destinations — reject anything not on the list
- For user-supplied URLs: show the destination clearly before redirecting
- Relative paths only for internal redirects
- Warn before navigating away from the secure origin

---

## 5. Social Engineering / Phishing

**What it is:** Attacker spoofs your trusted UI to trick users into divulging credentials
or executing malicious actions. Common forms: phishing emails with lookalike links,
spoofed login pages, fake security dialogs.

**UI attack surfaces:**
- Any UI that looks like a browser or OS dialog (custom permission prompts, auth flows)
- Links whose text doesn't match their actual destination
- Login pages loaded in third-party contexts
- "Download this file" prompts triggered without user gesture

**Mitigations:**
- Distinguish your trusted UI chrome clearly from user-generated content zones
- Never let user-generated content render in a way that can mimic your own UI
- Show actual destination URLs when link text differs from the URL
- Require explicit user gesture before launching downloads or running code
- Sandbox user-generated embeds: `<iframe sandbox="allow-scripts allow-same-origin">`
- Use signed email / clear sender verification for security-sensitive communications

---

## 6. Insecure Data Storage (Client-Side)

**What it is:** Secrets stored in browser storage (localStorage, sessionStorage) are
readable by any JavaScript on the page — including injected XSS scripts.

**Attack surfaces:**
- JWTs / session tokens in localStorage
- API keys hardcoded in frontend JS
- Passwords cached in JS variables

**Mitigations:**
- Session tokens → `HttpOnly; Secure; SameSite` cookies (not accessible to JS)
- Never hardcode secrets in frontend code
- Store only non-sensitive, non-auth data in localStorage
- Minimize what's kept in memory; clear sensitive values after use

---

## 7. Sensitive Data Exposure in UI

**What it is:** Sensitive information rendered in the UI, URLs, console logs, or
error messages can be harvested by attackers or leaked in browser history/logs.

**Attack surfaces:**
- Full credit card/SSN/account numbers displayed in UI
- Tokens or IDs in page URLs (browser history, referrer headers)
- Verbose error messages revealing stack traces, DB schemas, or file paths
- `console.log(userObject)` with sensitive fields

**Mitigations:**
- Mask sensitive values: `••••••1234`, never display full PANs
- Keep tokens/IDs out of URLs; use POST body or cookies
- Generic error messages for users; detailed errors logged server-side only
- Strip sensitive fields before logging objects to console

---

## 8. Tab-Napping (Opener Hijacking)

**What it is:** A page opened via `target="_blank"` gets a reference to the opener
window via `window.opener` and can redirect it to a phishing page.

**Attack surfaces:**
- Any `<a target="_blank">` link to external content
- `window.open()` calls to external URLs

**Mitigations:**
- Always add `rel="noopener noreferrer"` to external `target="_blank"` links
- Use `window.open(url, '_blank', 'noopener,noreferrer')` in JavaScript

---

## 9. Insecure Authentication UX

**What it is:** Auth UI that makes it easy for users to be deceived, creates weak
security by default, or fails to communicate security state clearly.

**Attack surfaces:**
- No visible HTTPS indicator / insecure fallback silently allowed
- "Remember me" checked by default on shared computers
- Ambiguous permission scopes ("this app needs access to your account")
- CAPTCHA-only auth without accessible alternatives
- Password fields that autocomplete on shared devices

**Mitigations:**
- Loudly indicate insecure connections; never silently degrade to HTTP
- "Stay logged in" is opt-in, not opt-out
- Express OAuth/permission scopes in plain language listing exact data accessed
- Provide accessible auth alternatives (magic link, passkey) alongside CAPTCHAs
- `autocomplete="current-password"` on login; `autocomplete="new-password"` on signup
- `autocomplete="one-time-code"` on OTP fields; `autocomplete="off"` for sensitive non-standard fields
- Prefer passwordless (WebAuthn/passkeys, magic links) — eliminates phishing and credential stuffing

---

## 10. Supply Chain / Third-Party Script Injection

**What it is:** A compromised CDN, npm package, or analytics script executes in your
page's context with full access to the DOM and user data.

**Attack surfaces:**
- CDN-hosted scripts without integrity checks
- npm packages with downstream vulnerabilities
- Analytics / tag manager scripts with broad scope

**Mitigations:**
- Subresource Integrity (SRI) on all external scripts:
  `<script src="..." integrity="sha384-HASH" crossorigin="anonymous">`
- Strict CSP to restrict script sources to known origins
- Regularly audit dependencies: `npm audit`, Dependabot, Snyk
- Minimize third-party script surface — each external script is a risk

---

## 11. Insecure File Handling UI

**What it is:** File upload/download UIs that don't communicate security implications
to users, or allow dangerous file types without appropriate warnings.

**Attack surfaces:**
- File save dialogs that allow saving to unprotected/shared locations
- Upload UIs that accept any file type without validation
- "Open attachment" prompts without type verification

**Mitigations:**
- Restrict save locations when content is sensitive; warn about unprotected paths
- Accept file uploads with explicit allowed types in the `accept` attribute
- Validate actual file content server-side (not just extension or MIME type)
- Warn before opening executable or potentially dangerous file types
- Use image rewriting libraries to strip extraneous content from uploaded images

---

## 12. Permission / Authorization UX Failures

**What it is:** Users grant broader or unintended permissions because the UI doesn't
make the scope, consequences, or permanence clear.

**Attack surfaces:**
- Vague permission dialogs ("Allow access to your account")
- No clear revocation path after a permission is granted
- Permissions that silently expand scope (granting one thing also grants another)
- No persistent UI showing that sharing/access is currently active

**Mitigations (from Apple Secure Coding Guide):**
- State consequences, not technical details: "Anyone with the link can edit this document"
- Show full scope before granting: "This also shares all files in the same folder"
- Provide a clear, easy revocation path — and warn that revoking doesn't undo past access
- Show a persistent indicator when sharing or elevated access is active
- Never make an insecure option the default — secure = default, insecure = opt-in
- For irreversible actions: make this explicit before confirmation; no default option

---

## 13. Homograph / IDN Spoofing

**What it is:** Attacker registers a domain that looks identical to a trusted domain
using Unicode characters that are visually indistinguishable from ASCII
(e.g., Cyrillic "а" vs Latin "a"). Users cannot tell the difference by looking.

**Attack surfaces:**
- Links to external sites shown in your UI
- User-supplied URLs rendered as clickable links
- OAuth/SSO login redirects to homograph domains

**Mitigations:**
- Display URLs in Punycode for domains containing mixed scripts: `http://www.xn--pple-43d.com`
- Validate redirect/callback URLs against an explicit allowlist of ASCII domains
- Warn users before navigating to domains outside your trusted set
- Use browser-native URL display wherever possible — don't roll your own URL parser

---

## Quick Threat-to-Mitigation Matrix

| Threat | Primary Defense | Secondary Defense |
|---|---|---|
| XSS | Framework auto-escaping | CSP header |
| CSRF | SameSite cookies | CSRF token |
| Clickjacking | `frame-ancestors 'none'` | X-Frame-Options |
| Open Redirect | Allowlist destinations | Show URL before redirect |
| Tab-Napping | `rel="noopener noreferrer"` | — |
| Token theft | HttpOnly cookie | No secrets in localStorage |
| Supply chain | SRI on scripts | Strict CSP |
| Social engineering | Trusted path / clear UI chrome | Sandbox user content |
| Phishing | Show actual URLs | Signed communications |
| Permission creep | Explicit scope + revocation UI | Least-privilege defaults |
