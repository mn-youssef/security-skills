---
name: authentication-testing
description: Use when testing login, sessions, password reset, OAuth/OIDC/SAML/SSO, MFA, and JWTs for weaknesses that lead to account takeover — credential attacks, session fixation, reset-token poisoning, MFA bypass, and token forgery. On apps you own or are authorized to test.
---

# Authentication Testing (FIND/EXPLOIT — OWASP A07)

## Overview
Break "are you who you say you are?" Account takeover (ATO) is the crown-jewel attack — one
auth flaw often unlocks everything. Test the whole identity lifecycle, not just the login form.

**Core principle:** Auth is a *flow*, not a page. Bugs live in reset, refresh, federation, and
state transitions — not just the password check.

## Applies when / Skip when
- **Applies when:** the app has any identity mechanism — login, sessions, accounts, tokens/JWT,
  OAuth/SSO, password reset, or MFA.
- **Skip when:** the app is entirely anonymous with no authentication → N/A.
- **If N/A:** report "authentication-testing: N/A — no authentication surface" and stop.

## ⚠️ Authorization
Use your own accounts/app. Do **not** credential-stuff real users or third parties. Brute-force
only your own test accounts, throttled.

## Surfaces to test (the whole lifecycle)
1. **Registration** — username/email enumeration, weak password policy, duplicate/normalization.
2. **Login** — rate limiting, lockout, error-message enumeration, default creds.
3. **Session** — token entropy, fixation (rotate on login?), invalidation on logout/reset, cookie flags.
4. **Password reset** — token entropy/expiry/reuse, **host-header poisoning** of reset links, user-id swap.
5. **MFA/2FA** — bypass via flow skip, response tampering, backup-code abuse, OTP brute/rate, race.
6. **OAuth/OIDC/SAML/SSO** — `redirect_uri` validation, `state`/CSRF, token leakage, SAML signature.
7. **JWT** — `alg:none`, algorithm confusion, weak secret, signature not verified, no expiry/revocation.

See `references/auth-attacks.md` for concrete recipes per surface.

## Quick reference — high-value bugs
| Bug | Test | Impact |
|-----|------|--------|
| User enumeration | Different response/timing for valid vs invalid user | Targeted attacks |
| No rate limit | Many login/OTP attempts succeed | Brute-force / ATO |
| Session fixation | Session id unchanged after login | Hijack |
| Reset poisoning | `Host`/`X-Forwarded-Host` controls reset link domain | ATO via stolen token |
| Reset id swap | Reset for victim using your token/flow | ATO |
| MFA bypass | Skip step / tamper `"mfa":false` / reuse code | ATO past 2FA |
| OAuth open redirect | `redirect_uri` to attacker domain leaks code/token | ATO |
| JWT forgery | `alg:none` / HS-RS confusion / weak secret accepted | Full impersonation |

## Output
Findings with the exact flow and request, which lifecycle stage, and whether it yields ATO.
Note chains (e.g. enumeration + no-rate-limit + weak reset = full ATO) for `vulnerability-chaining`.

## Hand-off
Reproduce on the running app via **`active-pentest`**; combine into ATO chains in
**`vulnerability-chaining`**; fix via **`security-hardening`** (bcrypt/argon2, CSPRNG tokens,
session rotation, strict `redirect_uri`, verified JWT signatures, rate limits + MFA).

## Common mistakes
- Testing only the login box — reset and federation flows are softer and just as fatal.
- Decoding a JWT and assuming it's verified — *prove* the signature is checked (tamper it).
- Treating MFA as unbreakable — most bypasses are logic flaws in the *flow*, not the crypto.
- Ignoring timing/error differences that quietly enable user enumeration.
