# Database Security Audit Reference

## SQL injection
**The classic sink:** a query built by string concatenation/interpolation with input.
```python
# VULNERABLE
cur.execute("SELECT * FROM users WHERE email = '" + email + "'")
db.query(f"SELECT * FROM orders WHERE id = {order_id}")
# SAFE — parameterized
cur.execute("SELECT * FROM users WHERE email = %s", (email,))
```
**Grep:** `execute(`, `query(`, `raw(`, f-strings/template literals/`+` inside SQL strings.

**Also check:**
- **Dynamic identifiers**: table/column/`ORDER BY` names from input can't be parameterized —
  must be allowlisted.
- **ORM raw escapes**: `Model.objects.raw()`, `sequelize.query()`, `knex.raw()`, `.whereRaw()`.
- **Second-order**: stored input later concatenated into a query.
- **Blind/time-based**: even endpoints with no visible output can be injectable.

## NoSQL injection
**MongoDB et al.:** operator injection when input is an object.
```js
// VULNERABLE — {"$ne": null} bypasses the check
db.users.find({ user: req.body.user, pass: req.body.pass })
```
**Check:** are inputs cast to expected types (string) before querying? `$where`/JS execution used?

## Broken access control at the data layer
- Queries scoped to the authenticated user? `WHERE user_id = :me` present, not just `WHERE id = :id`.
- Row-level security / tenant isolation enforced (multi-tenant `org_id`).
- Soft-deleted / archived rows excluded.

## Sensitive data exposure
- PII/secrets stored in plaintext that should be encrypted at rest.
- Passwords hashed with bcrypt/argon2 (never reversible/encrypted/md5).
- `SELECT *` returning columns (password hash, tokens, internal flags) the API then leaks.
- Backups/dumps/replicas with weaker access controls.
- Connection strings & DB creds hardcoded or in committed config.

## Least privilege
- App DB user has only needed grants (not `GRANT ALL` / DBA).
- Separate read/write users where it helps.
- No `DROP`/`ALTER`/`FILE` privileges for the app account.

## Quick grep starters
```bash
grep -rniE "execute\(|\.query\(|raw\(|whereRaw|f\"SELECT|f'SELECT|\\+ *['\"].*SELECT" .
grep -rniE "GRANT ALL|SELECT \*|password|secret|api[_-]?key" . --include=*.sql
```
