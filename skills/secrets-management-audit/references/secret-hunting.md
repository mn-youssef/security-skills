# Secret Hunting & Management Recipes

> Your own/authorized repos & infra. Found a real secret? Rotate first, then fix the leak path.
> Never *use* discovered third-party credentials.

## Scan source + full git history
```bash
trufflehog git file://. --only-verified         # verified = likely live (own repos)
gitleaks detect --source . --redact             # patterns + entropy, scans history
detect-secrets scan > .secrets.baseline         # baseline for ongoing diffs
```
History matters: a secret removed in a later commit is still in earlier ones.

## Quick grep patterns
```bash
grep -rniE "api[_-]?key|secret|passwd|password|token|private[_-]?key|client[_-]?secret" \
  --include=*.{js,ts,py,rb,go,java,env,yml,yaml,json,properties} .
grep -rniE "AKIA[0-9A-Z]{16}|ghp_[0-9A-Za-z]{36}|xox[baprs]-|-----BEGIN .*PRIVATE KEY-----" .
```
Common high-value formats: AWS `AKIA…`, GitHub `ghp_…`/`github_pat_…`, Slack `xox…`, Google
`AIza…`, Stripe `sk_live_…`, JWT, SSH/PGP private-key blocks, DB connection strings.

## Other surfaces
- **Frontend:** grep built bundles + source maps for keys (anything `sk_`/service keys ≠ publishable).
- **Mobile:** decompile the binary (`apktool`/`jadx`) and grep for keys/endpoints.
- **Logs:** grep app/CI/APM logs for tokens & PII; ensure logging redacts secrets.
- **Config/env/IaC:** `.env`, container env, Terraform state, cloud user-data.

## Validate (carefully)
Confirm a hit is live only for **your own** keys (e.g. a benign authenticated call you own). For
third-party-looking creds, do **not** use them — treat as exposed and rotate regardless.

## Remediate
1. **Rotate** the exposed secret immediately (it's compromised the moment it's committed).
2. **Purge from history** (so it's not re-clonable):
   ```bash
   git filter-repo --invert-paths --path path/to/secret-file   # or BFG --replace-text
   # force-push + have collaborators re-clone; rotate anyway (history may be cached)
   ```
3. **Move to a manager** — Vault / AWS Secrets Manager / GCP Secret Manager / Azure Key Vault;
   inject at runtime, never bake into code/images.
4. **Least privilege & rotation** — scope each secret minimally; set rotation; prefer short-lived
   tokens / OIDC over long-lived keys.

## Prevent (make it continuous)
- Pre-commit hook: `gitleaks protect` / `detect-secrets-hook` blocks secrets before commit.
- CI gate: run a secret scan on every PR/build; fail the build on new findings.
- Education: publishable vs secret keys; never paste secrets into code/chat/tickets.
