---
name: file-upload-and-ssrf
description: Use when testing file-upload handling, server-side request forgery (SSRF), and insecure deserialization — upload bypasses (type/extension/content, polyglots, path traversal), SSRF to cloud metadata and internal services, and unsafe deserialization sinks. On apps you own or are authorized to test.
---

# File Upload, SSRF & Deserialization (FIND/EXPLOIT — server input handling)

## Overview
Three high-impact classes that often reach RCE or full cloud compromise and were previously
homeless across the references: untrusted **files**, untrusted **URLs**, and untrusted
**serialized objects**. All three are "the server processes attacker-supplied data."

**Core principle:** Anywhere the server *fetches*, *stores+serves*, or *reconstructs* attacker
data, ask what else it can be pointed at or made into.

## Applies when / Skip when
- **Applies when:** the app accepts **file uploads**, **fetches user-supplied URLs**, or
  **deserializes** input — test each of the three independently.
- **Skip when:** a given feature is absent → that part is N/A (e.g. no uploads → skip the upload
  section but still test SSRF/deserialization if present).
- **If N/A:** report which of the three are absent and test only what exists.

## ⚠️ Authorization
Your own/authorized non-prod. SSRF can reach real internal infra and cloud creds — keep targets
to your environment; use benign canaries, don't pivot out of scope.

## File upload
| Check | Test |
|-------|------|
| Type/extension bypass | `shell.php.jpg`, double ext, null byte, case, `.phtml`/`.svg`/`.htaccess` |
| Content-type spoof | Real image bytes + script; mismatched `Content-Type` header |
| Polyglot | File valid as image **and** script (e.g. GIF/PHP) |
| Path traversal | Filename `../../var/www/shell.php` to control where it lands |
| Executable location | Does it land in a web-served, executable dir? |
| Parser attacks | Malicious SVG (XXE/XSS), Office/ZIP (zip-slip, XXE), image lib CVEs |
| Limits | Size/rate limits; decompression bombs |

The kill condition is **upload → reachable executable path → execution**. If files are stored
outside webroot with random names and non-executable, most upload bugs lose their teeth.

## SSRF
- **Sinks:** URL preview/unfurl, webhooks, import-from-URL, PDF/screenshot/image fetch, SSO
  metadata, file:// handlers, server-side HTTP clients.
- **Targets to prove (benign):** your collaborator/OOB host, then `127.0.0.1`/internal, then cloud
  metadata `http://169.254.169.254/latest/meta-data/` (AWS), `metadata.google.internal` (GCP).
- **Bypasses to test:** alternate IP encodings, `[::1]`, DNS rebinding, redirects, `@`-tricks,
  allowlist parsing flaws.
- SSRF→cloud-metadata→IAM creds is a top **chain** to demonstrate (see `vulnerability-chaining`).

## Insecure deserialization
- **Sinks:** `pickle.loads`, `yaml.load`, Java `readObject`, PHP `unserialize`, .NET
  `BinaryFormatter`, Node `node-serialize` on untrusted input.
- Detect: serialized blobs in cookies/params/bodies; tampering changes server behavior.
- Confirm with a benign gadget (OOB ping) — don't run destructive gadget chains.

See `references/upload-ssrf-recipes.md` for payloads and safe confirmation.

## Output
Per finding: sink, payload, evidence (where the file executed / OOB hit / object reconstructed),
and reachable impact (RCE, internal access, cloud creds). Usually High–Critical.

## Hand-off
Frequently the foothold/pivot in `vulnerability-chaining`; reproduce via `active-pentest`; fix via
`security-hardening` (store outside webroot + validate content + random names; URL allowlist +
block internal ranges + IMDSv2; safe deserialization formats).

## Common mistakes
- Checking only the file extension client-side — validate content server-side.
- Declaring "no SSRF" without testing redirects, DNS rebinding, and IP-encoding bypasses.
- Testing SSRF against real internal infra/prod — stay in your environment.
- Treating deserialization as theoretical — it's a common direct path to RCE.
