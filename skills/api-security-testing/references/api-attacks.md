# API Attack Recipes (REST · GraphQL · gRPC · WebSocket)

> Your own/authorized API. Test accounts A/B + low/admin. Non-prod for stateful tests.

## Discovery
```bash
# Specs & docs
curl -s URL/openapi.json | jq '.paths | keys'
curl -s URL/swagger /api-docs /v2/api-docs
# Endpoints from the app's own JS / mobile traffic (proxy)
# Old versions
for v in v1 v2 v3 internal beta; do curl -s -o /dev/null -w "%{http_code} $v\n" URL/api/$v/users; done
```

## REST
**Authorization (BOLA / BFLA / mass-assignment) is the same methodology on every surface — the
swap recipes live in `access-control-testing/references/access-control-tests.md`.** Apply them to
each endpoint you discovered above. Below are the checks *unique to APIs*:
```bash
# Excessive data exposure (API3) — API returns more than the UI shows; read the RAW json
curl -s URL/api/users/me -H "Authorization: Bearer $TOKEN_LOW" | jq
#   look for: passwordHash, tokens, internal flags, other users' fields the UI never renders
# Method tampering — a verb may not be authz-checked even when GET is
for m in GET POST PUT PATCH DELETE OPTIONS; do
  curl -s -o /dev/null -w "$m %{http_code}\n" -X $m URL/api/admin/config -H "Authorization: Bearer $TOKEN_LOW"
done
# Unrestricted resource consumption (API4) — missing size/page/rate limits
curl -s "URL/api/items?limit=1000000" -H "Authorization: Bearer $TOKEN_A"
```
Also API-specific: content-type confusion (send JSON as form / XML→XXE), wrong `Accept`,
parameter pollution. *(BOLA/BFLA/mass-assignment → `access-control-testing`.)*

## GraphQL
```graphql
# Introspection — full schema (disable in prod)
{ __schema { types { name fields { name } } } }
```
```bash
curl -s URL/graphql -H 'Content-Type: application/json' \
  -d '{"query":"{ __schema { queryType { name } types { name fields { name } } } }"}'
```
- **Batching / aliasing** (rate-limit & brute bypass — one HTTP request, many operations):
  ```graphql
  { a: user(id:1){email} b: user(id:2){email} c: user(id:3){email} }
  mutation { l1: login(u:"a",p:"1"){t} l2: login(u:"a",p:"2"){t} }  # password brute in one call
  ```
- **Depth/complexity DoS:** deeply nested recursive relations (`posts{author{posts{author{...}}}}`).
- **Field-level authz:** a sensitive field blocked on one type but reachable via another query/mutation/edge.
- **Mutations:** test authz on every mutation, not just queries.

## gRPC
- Enable reflection listing: `grpcurl -plaintext TARGET:PORT list` then `describe`.
- Call methods directly with crafted messages; test authz per-method (BFLA), field exposure (BOPLA).
- Check TLS, auth metadata validation, and that reflection is **off** in prod.

## WebSocket
- Does the WS handshake authenticate/authorize? Can you connect cross-origin (no origin check)?
- Send messages for actions you shouldn't reach; swap ids in messages (BOLA over WS).
- No rate limiting on the socket; message-flood / oversized frames.

## Misconfiguration & inventory
```bash
# CORS — does it reflect arbitrary origins w/ credentials?
curl -s -I URL/api/me -H "Origin: https://evil.com" | grep -i access-control
# Verbose errors
curl -s URL/api/users/'; -- ' | head            # provoke an error, read the leak
```
Flag: `Access-Control-Allow-Origin` reflecting `evil.com` + `Allow-Credentials: true`; stack
traces; old `/v1`; staging/debug APIs; missing `Cache-Control`/security headers.

## Map findings to the API Top 10
BOLA→API1, auth→API2, BOPLA→API3, no-limits→API4, BFLA→API5, flow-abuse→API6, SSRF→API7,
misconfig→API8, old-versions→API9, blind-trust-of-3rd-party→API10.
