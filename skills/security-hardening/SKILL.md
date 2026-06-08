---
name: security-hardening
description: Use when you have a security finding (from code audit or pentest) and need to fix it correctly with secure-coding patterns, then verify the fix actually closes it. Covers fixes for injection, access control, XSS, auth, secrets, and config across the stack.
---

# Security Hardening (FIX phase)

## Overview
Turn findings into correct, verified fixes. Fix the **root cause** at the right layer, add
**defense in depth**, then **re-test** to prove the hole is closed.

**Core principle:** A fix you didn't re-test is a hope, not a fix. Reproduce the original
exploit and confirm it now fails.

## Process per finding
1. **Root cause** — why is untrusted data reaching the sink? Fix *there*, not by blocklisting symptoms.
2. **Apply the secure pattern** — see table below.
3. **Defense in depth** — add a second independent control (validation + parameterization, etc.).
4. **Re-test** — rerun the exact PoC from `active-pentest`/the audit; confirm it fails now.
5. **Regression** — add a test so it can't silently come back.

## Secure-pattern quick reference
| Vulnerability | Root-cause fix | Defense in depth |
|---------------|----------------|------------------|
| SQL/NoSQL injection | Parameterized queries / ORM bindings; allowlist dynamic identifiers | Least-privilege DB user; input validation |
| Command injection | Avoid shell; use exec arrays / safe APIs; no shell interpolation | Allowlist args; drop privileges |
| XSS | Context-aware output encoding; framework auto-escape; `textContent` | CSP; sanitize HTML (DOMPurify); `HttpOnly` cookies |
| Broken access control / IDOR | Enforce authz server-side on every action; scope queries to the user | Deny-by-default; centralized policy checks |
| Mass assignment | Explicit allowlist of bindable fields (DTO/schema) | Separate read/write models |
| Broken auth | bcrypt/argon2; CSPRNG tokens; rotate session on login | MFA; rate limit; short token TTL + revocation |
| Crypto failures | Vetted libs; AES-GCM; TLS everywhere; no homemade crypto | Key management/rotation; HSTS |
| SSRF | Allowlist destinations; resolve+validate; block metadata/internal IPs | Egress firewall; no raw URL fetch |
| Insecure deserialization | Safe formats (JSON); `yaml.safe_load`; signed payloads | Type allowlists; isolation |
| Secrets in code | Move to env/secret manager; rotate the exposed secret | Secret scanning in CI; short-lived tokens |
| Security misconfig | Disable debug; least-priv CORS; remove default creds | Hardened baseline; config review in CI |
| Mobile insecure storage | Keychain/Keystore + encryption | Pinning; no secrets in binary |

## Validation rules of thumb
- **Validate on the server** — client validation is UX, not security.
- **Allowlist > blocklist** — define what's permitted, reject the rest.
- **Encode for the sink, not the source** — same data needs different encoding per context.
- **Fail closed** — deny by default; an error should not grant access.

## Re-test gate (don't skip)
```
[ ] Re-ran the original PoC → it now fails / is blocked
[ ] Added an automated regression test
[ ] Checked the fix didn't just move the sink elsewhere
[ ] Verified no similar instances exist (grep the pattern across the codebase)
```

## Output
For each finding: root cause · the fix (diff/pattern) · re-test result (PoC now fails) ·
regression test added. Update the findings register status → **Closed**.

## Common mistakes
- Patching the one reported line while identical sinks remain elsewhere — grep for siblings.
- Blocklist filters (e.g. stripping `<script>`) that attackers trivially bypass.
- Client-side-only fixes for server-side problems.
- Marking "fixed" without rerunning the exploit.
