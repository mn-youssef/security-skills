# STRIDE & Attack-Surface Reference

## Full-stack entry-point checklist
Walk every layer and list untrusted-input sources.

### Frontend (web)
- URL params, route params, hash/fragment
- Form fields, file uploads, drag-drop
- `postMessage`, WebSocket messages
- `localStorage`/`sessionStorage`/cookies read into the DOM
- Third-party scripts / embeds (supply chain)
- Client-side redirects (`window.location` from input)

### Backend API / server
- Every HTTP method on every route (don't forget `PUT`/`PATCH`/`DELETE`/`OPTIONS`)
- Headers: `Authorization`, `Cookie`, `Host`, `X-Forwarded-*`, `Referer`, custom headers
- Body: JSON, form-encoded, multipart, XML (→ XXE), GraphQL queries
- Query strings, path traversal in file params
- Webhooks & callbacks (often unauthenticated)
- Server-to-server calls, SSRF-reachable internal URLs
- Deserialization sinks, template rendering (SSTI)
- Rate-limit-free endpoints (login, OTP, password reset, search)

### Database
- Every place a query is built from input (SQL/NoSQL/ORM raw)
- Stored procedures, dynamic table/column names
- Search / filter / sort params (often string-concatenated)
- Mass-assignment into ORM models

### Mobile
- API endpoints the app calls (intercept with a proxy)
- Local storage: SQLite, plist, SharedPreferences, Keychain/Keystore
- Hardcoded secrets / API keys in the binary
- Deep links / intent handlers / URL schemes
- Certificate validation (pinning bypass)
- IPC: Android exported components, iOS URL schemes

## Trust boundaries (where bugs cluster)
```
[ Internet ]──▶[ CDN/WAF ]──▶[ Frontend ]──▶[ API gateway ]──▶[ Service ]──▶[ DB ]
                                  ▲                                  ▲
                              user→app                          app→data
                          (XSS, CSRF, authz)              (injection, IDOR)
```
Every arrow is a boundary. Untrusted data crossing into a more-trusted zone must be
**authenticated, authorized, and validated** at the boundary — not "later."

## STRIDE prompts per component
- **Spoofing** — How is identity proven? Can tokens be forged/replayed/stolen? Default creds?
- **Tampering** — Is input validated server-side? Are integrity checks present (signatures, HMAC)?
- **Repudiation** — Are security events logged immutably? Can logs be erased by the actor?
- **Information disclosure** — Verbose errors? Directory listing? PII in logs/responses? IDOR?
- **DoS** — Unbounded loops/queries? Missing rate limits? Zip bombs / large payloads?
- **Elevation of privilege** — Horizontal (other users' data) and vertical (user→admin) checks on every action?

## Risk scoring
`risk = likelihood (1–3) × impact (1–3)`
- Likelihood: how easy/discoverable is the attack? (3 = trivial, unauthenticated, remote)
- Impact: blast radius if it works? (3 = full account/data/system compromise)
- Triage: **6–9 fix now**, 3–4 plan, 1–2 backlog.
