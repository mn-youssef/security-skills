---
name: secrets-management-audit
description: Use when hunting for exposed secrets and auditing how an app you own manages them — API keys, tokens, passwords, and private keys in source, git history, binaries, logs, config, and CI; plus vaulting, rotation, and least-privilege of the secrets themselves.
---

# Secrets Management Audit (FIND/FIX — credentials & keys)

## Overview
Leaked secrets are among the fastest paths to compromise — one committed key can hand over a whole
cloud account. This skill finds exposed secrets everywhere they hide and audits whether the app
manages them properly (vaulted, rotated, scoped).

**Core principle:** A secret in source/history/logs/binary is **already public** — assume
compromised and rotate. Detection without rotation is theater.

## Applies when / Skip when
- **Applies when:** essentially always — every app has secrets/keys/config somewhere (source,
  git history, logs, env, build artifacts, CI).
- **Skip when:** effectively never. Even a static site has API keys, deploy creds, or CI tokens.
- **If genuinely nothing to scan:** say so explicitly — but verify first; "no secrets" is almost
  always "didn't look hard enough."

## ⚠️ Authorization
Your own/authorized repos, artifacts, and infrastructure. If you find a real exposed secret,
treat it as an incident: rotate first, then fix the leak path.

## Where secrets hide (hunt all of these)
| Location | How |
|----------|-----|
| Source code | Scanners + grep for key patterns |
| **Git history** | Secrets deleted from HEAD still live in history — scan all commits |
| Frontend bundles | JS/source maps shipping keys to the browser |
| Mobile binaries | Decompiled APK/IPA (`apktool`/`jadx`, `strings`) |
| Logs & error output | Tokens/PII printed to stdout/log files/APM |
| Config & env | `.env`, config files, container env, cloud user-data |
| CI/CD | Pipeline files, build logs, masked-but-leaked env vars |
| Cloud | Instance metadata, function env, IaC state, config |

See `references/secret-hunting.md` for tooling and patterns.

## Method
1. **Scan broadly** — full git history + working tree + artifacts + logs with entropy + pattern detectors.
2. **Validate** — is the hit a live credential? (don't *use* third-party creds; verify benignly/own-keys only)
3. **Assess blast radius** — what does the secret unlock? scope/permissions?
4. **Manage audit** — are secrets vaulted, rotated, least-privilege, not in code?
5. **Remediate** — rotate exposed secrets, purge from history, move to a manager, add prevention.

## Tooling
`trufflehog`, `gitleaks`, `detect-secrets` (detection incl. history & entropy); `git filter-repo`/
BFG (purge history); cloud secret managers / Vault (storage); pre-commit hooks + CI scanning (prevent).

## Output
Per finding: secret type, location (incl. commit hash), validity, blast radius, and rotation status.
Live high-privilege keys are Critical — rotate immediately.

## Hand-off
Chains (leaked key → account/cloud takeover) → `vulnerability-chaining`; fix & prevention →
`security-hardening` (rotate, vault, scope, pre-commit/CI secret scanning).

## Common mistakes
- Scanning only HEAD — **git history** is where deleted secrets live forever.
- Removing a secret from code but **not rotating** it — it's already compromised.
- Shipping "public" keys that are actually secret/service keys in frontend/mobile bundles.
- No prevention — without pre-commit/CI scanning, the next secret lands next week.
