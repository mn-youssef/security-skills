---
name: security-code-audit
description: Use when you have application source code and want to find security vulnerabilities by reading it — covers OWASP Top 10 across frontend, backend API, and database. Produces a findings register with severity, file:line, and exploitability.
---

# Security Code Audit (FIND phase)

## Overview
Read source code and find vulnerabilities by pattern. No running app required — works in any
AI coding tool. Trace untrusted input from its **source** (where it enters) to its **sink**
(where it's used dangerously).

**Core principle:** A vulnerability is *untrusted data reaching a dangerous sink without
adequate validation/encoding in between*. Find the source→sink paths.

## Method: source → sink tracing
1. From the threat model (or by scanning entry points), pick a **source** of untrusted input.
2. Follow the data through the code to where it's **used** (the sink).
3. Ask: is it validated, sanitized, parameterized, or encoded *for that sink*? If not → finding.
4. Record severity, `file:line`, the exact data path, and how to exploit it.

## OWASP Top 10 — what to grep for
| Risk | Smell / sink to search |
|------|------------------------|
| Injection (SQL/NoSQL/cmd) | string-built queries, `exec`, `eval`, `system`, template interpolation in queries |
| Broken access control (IDOR) | object lookups by user-supplied id with no ownership check |
| Broken auth | custom crypto, weak session handling, missing MFA, token in URL |
| Cryptographic failures | `md5`/`sha1` for passwords, hardcoded keys, `Math.random` for tokens, no TLS |
| SSRF | server fetches a user-supplied URL (`requests.get(input)`, `fetch(input)`) |
| Security misconfig | debug=true, default creds, permissive CORS `*`, verbose errors |
| XSS | `innerHTML`, `dangerouslySetInnerHTML`, `v-html`, unescaped template output |
| Insecure deserialization | `pickle.loads`, `yaml.load`, native deserialize on input |
| Vulnerable dependencies | check lockfiles vs advisories (`npm audit`, `pip-audit`, `osv`) |
| SSTI / path traversal | template render of input, `../` in file paths from input |

## Domain references (load the ones that match your stack)
- `references/web-frontend.md` — XSS, CSRF, CSP, client-side secrets, clickjacking
- `references/api-backend.md` — authz/IDOR, injection, SSRF, JWT, rate limiting, mass assignment
- `references/database.md` — SQLi/NoSQLi, access control, data exposure, ORM pitfalls

## Output: findings register
| ID | Severity | Title | Location | Source→sink | Exploitability | Fix pointer |
|----|----------|-------|----------|-------------|----------------|-------------|
| F1 | High | SQLi in user search | `api/users.py:42` | `?q=` → raw SQL | Unauth, dumps DB | parameterize → `security-hardening` |

Severity = the threat-model risk score, adjusted for how exploitable the code actually is.

## Hand-off
High-severity findings on a running app → **`active-pentest`** to validate.
All findings → **`security-hardening`** to fix.

## Common mistakes
- Pattern-matching sinks without confirming input is actually attacker-controlled (false positive).
- Stopping at the first sink — one source often reaches several.
- Trusting client-side validation — it's bypassable; the server must validate.
- Ignoring authorization: most real-world breaches are *access control*, not exotic injection.
