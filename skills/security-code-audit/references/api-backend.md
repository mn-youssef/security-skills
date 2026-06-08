# Backend API / Server Security Audit Reference

> **Scope: static (white-box) code review — what to *read* and *grep* for in source.** To *test*
> these on a running app (black/grey-box), use the dynamic specialists: `access-control-testing`,
> `authentication-testing`, `api-security-testing`. This file finds the smell in code; they prove
> the exploit. No duplication — different method, same bug.

## Broken access control (the #1 real-world bug)
*Dynamic testing of these → `access-control-testing`. In code, look for:*
- **IDOR / horizontal**: `GET /api/orders/:id` that loads by id without checking the row
  belongs to the caller. **Grep:** `findById(req.params.id)` with no `where userId =`.
- **Vertical / privilege escalation**: admin actions guarded only in the UI, or a role check
  that's missing on some routes. Check **every** mutating endpoint enforces authz server-side.
- **Mass assignment**: binding the whole request body into a model lets a user set `isAdmin`,
  `role`, `balance`. **Grep:** `Model(**req.body)`, `Object.assign(user, req.body)`, `update(req.body)`.

## Injection
- **SQL/NoSQL**: see `database.md`.
- **Command**: `exec`, `spawn`, `system`, `os.popen`, backticks with input → RCE.
- **SSTI**: rendering a template from user input (`render_template_string(input)`).
- **Header/CRLF**: input reflected into response headers.
- **LDAP/XPath/XXE**: XML parsers with external entities enabled (`resolve_entities=True`).

## SSRF (Server-Side Request Forgery)
Server fetches a user-controlled URL. **Grep:** `requests.get(url)`, `fetch(url)`, `axios(url)`,
`urllib`, image/PDF/webhook fetchers, URL preview features.
**Check:** is the target validated against an allowlist? Is access to `169.254.169.254`
(cloud metadata), `localhost`, and internal ranges blocked?

## Authentication & sessions
*Dynamic testing (ATO, reset poisoning, OAuth/JWT/MFA) → `authentication-testing`. In code, look for:*
- Passwords hashed with **bcrypt/argon2/scrypt** (not md5/sha1/plain).
- Tokens generated with a CSPRNG (not `Math.random`/timestamp).
- Session fixation: is the session id rotated on login?
- Logout actually invalidates server-side. Token expiry enforced.
- Brute-force protection on login/OTP/reset (see rate limiting).

## JWT pitfalls
- `alg: none` accepted? Algorithm confusion (RS256 verified as HS256 with public key as secret)?
- Signature actually verified, not just decoded?
- Sensitive data in the (readable) payload? Long or missing expiry? No revocation?

## Rate limiting & resource limits
**Check for absence on:** login, signup, OTP, password reset, search, expensive queries,
file upload, GraphQL (query depth/complexity). Unbounded pagination/`limit` params.

## Security misconfiguration
- Debug mode / stack traces in production responses.
- CORS: `Access-Control-Allow-Origin: *` **with** `Allow-Credentials: true`, or reflecting
  arbitrary `Origin`.
- Default credentials, exposed actuator/admin/metrics endpoints.
- Verbose error messages leaking stack/SQL/paths.

## Insecure deserialization
**Grep:** `pickle.loads`, `yaml.load` (use `safe_load`), `marshal`, Java `readObject`,
PHP `unserialize`, `.NET BinaryFormatter` on untrusted input.

## Quick grep starters
```bash
grep -rniE "req\.(params|query|body)|request\.(args|form|json)" --include=*.js --include=*.py
grep -rniE "exec\(|system\(|popen|subprocess|child_process" .
grep -rniE "requests\.(get|post)|fetch\(|axios\(" .   # SSRF candidates
grep -rniE "jwt\.decode|verify=False|algorithms?=\[?'?none" .
```
