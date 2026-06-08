# Access Control Test Recipes

> Owned/authorized app. Set up: **Account A**, **Account B** (same level), **Low** (limited),
> **Admin**. Capture `$TOKEN_A`, `$TOKEN_B`, `$TOKEN_LOW`.

## IDOR / BOLA — object reference swap
```bash
# A's own object (baseline, should work)
curl -s -i URL/api/invoices/1001 -H "Authorization: Bearer $TOKEN_A"
# A requests B's object id, with A's token  -> if 200 + B's data = IDOR
curl -s -i URL/api/invoices/1002 -H "Authorization: Bearer $TOKEN_A"
```
Reference forms to try: integer ids (±1, enumerate), UUIDs (leaked elsewhere), slugs,
filenames, base64/hex-encoded ids, composite keys, ids in JSON body or headers (not just URL).

## Horizontal — act on a peer's resource
```bash
# A tries to UPDATE/DELETE B's resource
curl -s -i -X PUT URL/api/profile/B_ID -H "Authorization: Bearer $TOKEN_A" \
     -H 'Content-Type: application/json' -d '{"email":"attacker@evil.com"}'
```

## Vertical — low privilege reaches admin
```bash
# Low-priv account hits an admin function directly (skip the UI)
curl -s -i URL/api/admin/users -H "Authorization: Bearer $TOKEN_LOW"
curl -s -i -X POST URL/api/admin/promote -H "Authorization: Bearer $TOKEN_LOW" \
     -d '{"user":"me"}'
```

## BFLA — broken function-level authz
The endpoint exists, UI hides the button, but the function isn't access-checked.
Enumerate admin/privileged verbs and call them with a normal token. Also try method swap:
if `GET /admin/x` is blocked, try `POST`/`PUT`/`HEAD`/`OPTIONS`.

## Multi-tenant isolation
```bash
# Swap tenant/org scoping — does tenant A see tenant B?
curl -s URL/api/reports -H "Authorization: Bearer $TOKEN_A" -H "X-Org-Id: B_ORG"
curl -s URL/api/reports?org_id=B_ORG -H "Authorization: Bearer $TOKEN_A"
```

## Mass assignment / parameter pollution
```bash
# Add fields the client never sends
curl -s -X POST URL/api/users -H "Authorization: Bearer $TOKEN_LOW" \
  -H 'Content-Type: application/json' \
  -d '{"name":"x","email":"x@x.com","role":"admin","isVerified":true,"balance":99999}'
```
Also try duplicate params (`?role=user&role=admin`) and nested overrides (`user[role]=admin`).

## Forced browsing
- Enumerate unlinked routes: `/admin`, `/api/internal`, `/api/v1/admin`, `/debug`, `/metrics`.
- Diff what each role *can* see; anything reachable by a lower role = finding.
- Check static/exported files: `/exports/`, `/uploads/<other-user-file>`.

## Edge cases that hide bugs
- Authz checked on the **list** endpoint but not the **detail** endpoint (or vice-versa).
- Authz on the primary action but not on **bulk**/**export**/**search**/**GraphQL** equivalents.
- Authz enforced on the web app but missing on the **mobile/API** path to the same data.
- Soft-deleted/archived objects still readable by id.
- State-changing **GET**s (`/transfer?to=...&amount=...`) bypassing CSRF + authz assumptions.

## What "fixed" looks like (for the retest)
Every request above returns **403/404** for the unauthorized actor, the check is **server-side**,
scoped to the authenticated principal (`WHERE owner = :me`), and **deny-by-default**.
