# Web Frontend Security Audit Reference

## XSS (Cross-Site Scripting)
**Sinks to grep:** `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`,
`dangerouslySetInnerHTML` (React), `v-html` (Vue), `[innerHTML]` (Angular bypass),
`eval`, `setTimeout(string)`, `new Function`, jQuery `.html()`/`.append(userInput)`.

- **Reflected**: input from URL/params rendered into the page unescaped.
- **Stored**: input persisted, later rendered to other users.
- **DOM**: client reads `location.hash`/`search` and writes to a sink.

**Check:** is output encoded for its context? HTML body ≠ attribute ≠ JS ≠ URL ≠ CSS.
Framework auto-escaping (JSX `{}`, Vue `{{ }}`) is safe *until* you use the raw-HTML escape hatch.

**Fix direction:** prefer text APIs (`textContent`); if HTML needed, sanitize with DOMPurify; set CSP.

## CSRF (Cross-Site Request Forgery)
State-changing requests (POST/PUT/DELETE) that rely only on cookies are vulnerable.
**Check:** anti-CSRF token present and verified? `SameSite=Lax/Strict` on session cookies?
Is a custom header required (not sendable cross-origin without CORS)?

## Content Security Policy
**Check for absence or weakness:** no CSP header; `script-src 'unsafe-inline'` or `'unsafe-eval'`;
wildcard `*` sources; missing `object-src 'none'`, `base-uri`, `frame-ancestors`.

## Client-side secrets
Anything shipped to the browser is public. **Grep for:** API keys, tokens, private keys,
secrets in JS bundles, `.env` values inlined at build, hardcoded credentials.
Public/publishable keys are fine; secret/service keys are not.

## Clickjacking
**Check:** `X-Frame-Options: DENY/SAMEORIGIN` or CSP `frame-ancestors` set for sensitive pages.

## Open redirect
`window.location = param` or server redirect to a user-supplied URL → phishing.
**Check:** redirect targets validated against an allowlist.

## Sensitive data handling
- Tokens in `localStorage` are readable by any XSS. Prefer `HttpOnly` cookies for session tokens.
- PII rendered in the DOM/console/URLs.
- Autocomplete on sensitive fields, caching of authenticated pages.

## Quick grep starters
```bash
grep -rniE "innerHTML|dangerouslySetInnerHTML|v-html|document.write|eval\(|new Function" src/
grep -rniE "api[_-]?key|secret|token|private[_-]?key|password\s*=" src/ --include=*.js --include=*.ts
```
