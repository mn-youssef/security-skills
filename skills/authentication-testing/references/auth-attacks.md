# Authentication Attack Recipes

> Your own app/accounts. Throttle. Never target real users or third-party IdPs.

## User enumeration
- Compare responses for valid vs invalid usernames on: login, registration, reset, resend-OTP.
- Look at: error text ("no such user" vs "wrong password"), HTTP status, response length, **timing**.
- Fix shape: identical generic message + constant-time + same status for all cases.

## Login brute-force / credential attacks
- Confirm rate limiting & lockout exist (test on your accounts, throttled).
- Bypass tricks attackers use (so you can defend): rotating IP/`X-Forwarded-For`, casing/unicode
  variants of username, null bytes, lockout only per-account not per-IP (or vice-versa).
- Check default/seeded creds (`admin:admin`, demo accounts) left enabled.

## Session management
```bash
# Fixation: capture session id pre-login, log in, compare — must change
# Cookie flags: must be HttpOnly, Secure, SameSite for session cookies
curl -s -i URL/login -c jar.txt ...      # inspect Set-Cookie
```
- Logout invalidates server-side (reuse old token after logout → should fail).
- Password change/reset invalidates all other sessions.
- Token entropy: long, random (not sequential/timestamp/guessable).

## Password reset
- **Token quality:** length, randomness, single-use, short expiry, bound to the user.
- **Host-header poisoning:** if the reset email link is built from the request `Host`/
  `X-Forwarded-Host`, an attacker sets it to capture the victim's token:
  ```bash
  curl -s -X POST URL/reset -d 'email=victim@x.com' -H 'Host: attacker.com'
  # if the email links to attacker.com/reset?token=... the token leaks
  ```
- **User binding:** can you request a reset for victim and complete it via your own session/flow?
- **Token leakage:** token in URL → referrer/logs; reusable after use; not expired on use.

## MFA / 2FA bypass (logic flaws, not crypto)
- Skip the MFA step: go straight to the post-MFA endpoint with the half-authenticated session.
- Tamper the response: `{"mfaRequired":true}` → `false`, or change a redirect.
- OTP: brute (no rate limit), reuse, no expiry, race condition (parallel submits), backup-code abuse.
- "Remember device" token forgeable/predictable.

## OAuth / OIDC / SAML / SSO
- **`redirect_uri`:** exact-match enforced? Try attacker domain, path traversal, `//evil.com`,
  appended params, open-redirect chains to exfiltrate `code`/token.
- **`state`:** present and verified? Missing = CSRF on the OAuth flow.
- **Token leakage:** access/id tokens in URL fragments, logs, referrers.
- **SAML:** signature actually verified? XML signature wrapping (XSW)? `assertion` swap?
  Unsigned/improperly-scoped assertions accepted?
- **Account linking:** can you link your social login to a victim's local account (pre-account-takeover)?

## JWT attacks
```bash
# Decode (note: decode != verify)
echo "$JWT" | cut -d. -f2 | base64 -d 2>/dev/null
```
Test that the server **rejects**:
- `alg: none` (strip signature) — forge any claims.
- **Algorithm confusion**: token signed HS256 using the RS256 *public key* as the HMAC secret.
- **Weak secret**: crack HS256 with `hashcat -m 16500` against a wordlist (your own token).
- **No signature check**: tamper a claim (`"role":"admin"`), keep/garble signature — accepted?
- **Missing expiry / no revocation**: old/expired token still works.
- **`kid`/`jku`/`x5u` injection**: header points the verifier at attacker-controlled keys.

## Account-takeover chains (note these for vulnerability-chaining)
- enumeration → valid users → no-rate-limit → password spray → ATO
- reset poisoning OR reset-token reuse → direct ATO
- OAuth `redirect_uri` flaw → steal `code` → ATO
- JWT forgery (`alg:none`) → impersonate any user/admin
