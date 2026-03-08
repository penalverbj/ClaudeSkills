---
name: secure-ui
description: >
  Design and implement UI/UX with security built in from the start. Use this skill
  whenever building, modifying, or reviewing any user interface, form, authentication
  flow, input field, modal, navigation, link, file upload, data display, or interactive
  component — even if the user doesn't mention "security" explicitly. Apply secure
  defaults, anti-XSS patterns, CSRF protection, safe storage practices, social engineering
  resistance, and proper trust/permission UX. Always treat the UI as a security surface,
  not just a visual layer. Trigger for any frontend code, UI design decision, auth screen,
  error message, permission dialog, or user-facing data handling task.
---

# Secure UI/UX Design Skill

This skill ensures every UI component and user interaction Claude designs or implements
follows security best practices drawn from Apple's Secure Coding Guide, OWASP, and modern
frontend security standards.

**Core principle:** The user interface is a security surface. Attackers exploit UI as much
as code. The safest UI is one where secure behavior is the default, not an opt-in.

**Reference:** See `references/secure-ui-threats.md` for a full threat catalog with mitigations.

---

## Core Workflow

When writing or reviewing any UI code or design:

1. **Apply the secure UI checklist** for the component type (form, auth, nav, data display, etc.)
2. **Default to secure** — if something can be opt-in or opt-out, make the secure option the default
3. **Annotate security choices** inline (e.g., `// noopener prevents opener hijacking`)
4. **Flag risks** at the end of every response with the threat and recommended fix
5. **Never silently degrade** — if a secure option can't be applied, say so and give the user a choice

---

## The Secure UI Principles (Apple / Ka-Ping Yee)

These are the foundational properties every secure interface must give users:

| Property | What It Means in Practice |
|---|---|
| **Explicit Authorization** | Nothing becomes unsafe without the user actively choosing it |
| **Visibility** | Security state (SSL, sharing, permissions) is always clearly shown |
| **Revocability** | Permissions and access grants can always be undone |
| **Path of Least Resistance** | The easiest action is the safest action |
| **Expected Ability** | Users can always tell what they can do in the system |
| **Appropriate Boundaries** | Users can distinguish their data from others' |
| **Clarity** | Users understand what they are telling the system to do |
| **Identifiability / Trusted Path** | The system protects users from being impersonated or spoofed |

---

## Security Checklist by Component

### 📝 Forms & Input Fields

- [ ] Client-side validation for UX; **server-side validation is the security control**
- [ ] Use allowlist validation (define what IS valid), not denylist
- [ ] Never trust or render raw user input without sanitization
- [ ] `autocomplete` attributes set correctly (`autocomplete="off"` for OTP/sensitive fields)
- [ ] Sensitive fields (passwords, card numbers) use `type="password"` / `type="tel"` — never `type="text"`
- [ ] Avoid `innerHTML`, `document.write`, `eval()` with any user-supplied data — use `textContent` or framework-safe rendering
- [ ] Error messages describe what's wrong without revealing system internals (no stack traces, DB errors)
- [ ] Don't require users to re-enter data already submitted in the same session (also a UX win)
- [ ] File uploads: validate type server-side, never trust `Content-Type` header alone, restrict accepted extensions

### 🔐 Authentication & Permissions

- [ ] SSL/HTTPS connection indicator is prominent; insecure connections are **loudly flagged**, not silently allowed
- [ ] Secure defaults: sharing off, public access off, elevated permissions off until explicitly granted
- [ ] Show the current security/sharing state persistently (not just at the moment of change)
- [ ] Make the scope of permissions crystal clear before granting them (e.g., "this also shares all files in the folder")
- [ ] Every permission grant must be **revocable** — provide a clear revoke UI
- [ ] Warn users when an action cannot be undone **before** they take it
- [ ] Don't use cognitive function tests (CAPTCHAs) without an accessible alternative
- [ ] No passwords or tokens transmitted in cleartext or URLs
- [ ] Use `SameSite=Strict` or `SameSite=Lax` cookies; `HttpOnly` for session cookies
- [ ] Prefer passwordless auth (magic links, WebAuthn/passkeys) where feasible

### 🛡️ XSS Prevention

- [ ] Use framework-native rendering (React JSX, Vue templates, Angular interpolation) — these escape by default
- [ ] **Never** use `dangerouslySetInnerHTML` / `v-html` / `[innerHTML]` with untrusted data
- [ ] If rendering rich HTML is required, sanitize with an established library (e.g., DOMPurify)
- [ ] Avoid placing user data in: `href="javascript:..."`, `onclick="..."`, `<script>`, CSS `expression()`
- [ ] Set a Content Security Policy (CSP) header — see snippet below
- [ ] External scripts use Subresource Integrity (SRI): `<script integrity="sha256-..." crossorigin="anonymous">`
- [ ] `rel="noopener noreferrer"` on all `target="_blank"` links

### 🚫 CSRF Protection

- [ ] All state-changing requests (POST/PUT/PATCH/DELETE) carry a CSRF token
- [ ] Use framework-native CSRF protection if available (don't roll your own)
- [ ] CSRF tokens: unique per session, generated server-side, never in a cookie, never in a URL
- [ ] `SameSite` cookie attribute is set (works as a CSRF defense for modern browsers)
- [ ] Never trigger state-changing actions via GET requests

### 💾 Data Storage & Exposure

- [ ] **Never** store secrets, tokens, or credentials in `localStorage` or `sessionStorage` (XSS-accessible)
- [ ] Store session tokens in `HttpOnly; Secure; SameSite` cookies instead
- [ ] Don't log sensitive user data to the browser console
- [ ] Mask sensitive data in UI (e.g., `••••••1234` for card numbers, never show full PAN)
- [ ] Don't expose user IDs, session tokens, or internal references in URLs
- [ ] Use HTTPS everywhere; enforce with `Strict-Transport-Security` (HSTS) header

### 🔗 Links & Navigation

- [ ] All external links: `rel="noopener noreferrer"` to prevent tab-napping
- [ ] Validate redirect URLs against an allowlist — never redirect to user-supplied URLs unchecked
- [ ] Display the actual destination URL before navigation when it differs from link text
- [ ] Warn users before leaving a secure context (HTTPS → HTTP)
- [ ] Don't use `javascript:` URIs in `href`

### 🗂️ File Locations & Data Sensitivity

- [ ] Warn users when saving sensitive files to unprotected locations
- [ ] Make clear when a file or resource will be accessible to others
- [ ] If a user's choice reduces security (e.g., saving to a public folder), flag this prominently before confirming

### ⚠️ Dialogs, Warnings & Permission Prompts

- [ ] Express choices in terms of **consequences**, not technical details
  - ✅ "Anyone on the internet can view this file"
  - ❌ "This sets the ACL to public-read"
- [ ] The default option in a dangerous dialog is always the **safe** choice; if there's no safe choice, there's no default
- [ ] Don't show so many warnings that users start ignoring them — reserve dialogs for genuinely important decisions
- [ ] Security state must be indicated by **presence of a positive signal** (a lock when secure), not just absence of a warning
- [ ] Never surprise users with the results of an action they took

### 🎣 Social Engineering Resistance

- [ ] Clearly distinguish your UI's trusted elements from content that could be injected or user-generated
- [ ] Validate and display actual destination URLs — don't let link text masquerade as a different URL
- [ ] Warn users about homograph/IDN spoofing risks when displaying external domains
- [ ] Never automatically run code that the user hasn't explicitly authorized
- [ ] Sandbox user-generated content (iframes, embeds) with `sandbox` attribute
- [ ] Don't open external URLs in ways that could spoof your own trusted UI

---

## Code Snippets: Secure Patterns to Always Use

### Content Security Policy (HTTP header or meta tag)
```html
<!-- Strict CSP — adjust sources to match your app -->
<meta http-equiv="Content-Security-Policy"
  content="default-src 'self';
           script-src 'self' https://cdn.trusted.com;
           style-src 'self' 'unsafe-inline';
           img-src 'self' data: https:;
           connect-src 'self' https://api.yourapp.com;
           frame-ancestors 'none';">
```

### Safe external links
```html
<!-- Always add noopener noreferrer to external target="_blank" links -->
<a href="https://external.com" target="_blank" rel="noopener noreferrer">
  Visit site
</a>
```

### Safe user-content rendering (React)
```jsx
import DOMPurify from 'dompurify';

// ✅ Safe — framework escapes by default
<p>{userContent}</p>

// ✅ Safe when rich HTML is unavoidable — sanitize first
<div dangerouslySetInnerHTML={{
  __html: DOMPurify.sanitize(userHtmlContent)
}} />

// ❌ Never do this
<div dangerouslySetInnerHTML={{ __html: userContent }} />
```

### Secure cookies (server-side)
```
Set-Cookie: session=TOKEN; HttpOnly; Secure; SameSite=Strict; Path=/
```

### Subresource Integrity for CDN scripts
```html
<script
  src="https://cdn.example.com/lib.min.js"
  integrity="sha384-HASH_HERE"
  crossorigin="anonymous">
</script>
```

### Clickjacking prevention
```
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none';
```

### Open redirect prevention (JS)
```js
// ✅ Validate redirect targets against an allowlist
const ALLOWED_REDIRECTS = ['/dashboard', '/profile', '/settings'];

function safeRedirect(destination) {
  if (ALLOWED_REDIRECTS.includes(destination)) {
    window.location.href = destination;
  } else {
    window.location.href = '/'; // fallback to safe default
  }
}
```

---

## Security UX Language Guide

When writing copy for security dialogs, warnings, or permission prompts, translate
technical terms into user-facing consequences:

| Technical term | User-facing language |
|---|---|
| "Public ACL / world-readable" | "Anyone on the internet can see this" |
| "SSL/TLS certificate error" | "This site may not be who it claims to be — your information could be at risk" |
| "Insecure connection" | "Your information is not encrypted on this page" |
| "Grant OAuth scope `read:contacts`" | "This app will be able to read all your contacts" |
| "Revoking token" | "Removing this app's access to your account" |
| "Persistent cookie" | "Stay logged in on this device" |
| "Third-party cookie" | "This lets other websites track your activity here" |

---

## Threat Model Quick Reference

For each UI component, ask:
1. **What can the user input?** → Validate + sanitize
2. **What can the user see?** → Ensure sensitive data is masked or restricted
3. **What can the user trigger?** → Require explicit authorization; confirm destructive actions
4. **What does the user trust?** → Don't let untrusted content impersonate trusted UI

---

## What to Append to Every UI Response

After writing any UI code, add a **Security Notes** section:

```
## Security Notes

✅ Applied: [list key security choices made]
⚠️ Risk: [any gaps the caller must close, e.g. server-side validation, CSP headers]
🔒 Hardening tip: [optional improvement, e.g. add SRI, move to HttpOnly cookie]
```

---

## Reference Files

- `references/secure-ui-threats.md` — Full threat catalog: attack types, UI attack surfaces,
  and mitigations. Read when you need detail on a specific threat (XSS, CSRF, clickjacking,
  phishing, open redirect, etc.)
