---
name: recon-and-osint
description: Use when starting an assessment and you need to discover an application's full attack surface — subdomains, hosts, endpoints, technologies, exposed services, leaked credentials and secrets — on assets you own or are authorized to test. Feeds the threat model with real, not assumed, targets.
---

# Recon & OSINT (RECON phase — the front of the lifecycle)

## Overview
Find what's actually exposed before deciding what to test. Threat modeling assumes a surface;
recon *discovers* it. Most breaches start at an asset the defender forgot they had.

**Core principle:** You can't protect what you don't know exists. Enumerate everything first.

## ⚠️ Authorization
Only enumerate assets you own or are explicitly authorized to test. Passive OSINT on your own
org is always safe; active scanning/brute-forcing requires the asset to be in scope.

## Method (passive → active)
1. **Passive footprint** (no packets to target) — search engines, certificate transparency,
   public code, breach data, DNS records, archived pages. Zero noise, often highest value.
2. **Asset discovery** — subdomains, IP ranges, cloud buckets, related domains, forgotten staging.
3. **Service & tech fingerprinting** — what's running, which versions, which frameworks/CMS.
4. **Content discovery** — hidden endpoints, directories, API routes, JS-referenced URLs, params.
5. **Secret & exposure hunting** — leaked keys/tokens in public repos, paste sites, JS bundles, archives.
6. **Consolidate** — turn findings into a target inventory the threat model can rank.

See `references/recon-playbook.md` for concrete tools and commands per step.

## Quick reference — what to hunt
| Category | Goldmine sources |
|----------|------------------|
| Subdomains | crt.sh / cert transparency, `subfinder`, `amass`, DNS brute-force |
| Forgotten assets | staging/dev/old subdomains, S3/GCS/Azure buckets, archived pages (Wayback) |
| Tech stack | `whatweb`, `wappalyzer`, response headers, favicon hash, JS libs |
| Endpoints/params | `gau`/`waybackurls`, JS file parsing, `katana`, robots/sitemap, Swagger/GraphQL |
| Leaked secrets | GitHub/Gist dorking, `trufflehog`/`gitleaks` on public repos, paste sites |
| Exposed creds | breach-corpus lookups for org emails (defensive: know what attackers know) |

## Output: target inventory
A list the threat model consumes:

| Asset | Type | Tech | Notable | Exposure |
|-------|------|------|---------|----------|
| `api.example.com` | API host | Node/Express | Swagger exposed | public |
| `staging.example.com` | web | old build | debug=on | should be internal |

## Hand-off
Feed the inventory into **`threat-modeling`** to rank, then **`security-code-audit`** /
**`api-security-testing`** for the prioritized targets.

## Common mistakes
- Jumping to scanning before passive recon — you make noise and miss the easy wins.
- Enumerating only the main domain — forgotten subdomains/buckets are where breaches live.
- Ignoring JS bundles — they leak endpoints, params, and sometimes keys.
- Testing assets you found but aren't authorized for — discovery ≠ permission.
