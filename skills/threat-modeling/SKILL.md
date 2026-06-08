---
name: threat-modeling
description: Use when starting a security assessment and you need to map an application's attack surface, enumerate trust boundaries, and rank what to test first — before auditing code or running tools. Produces a prioritized target list.
---

# Threat Modeling (PLAN phase)

## Overview
Decide *what to worry about and test first* before spending effort. Map assets, entry
points, and trust boundaries; apply STRIDE; rank risks by likelihood × impact.

**Core principle:** You can't test everything. Spend effort where a break hurts most.

## Process
1. **Inventory assets** — What's valuable? (user data, money, credentials, PII, admin control, IP)
2. **Map entry points** — Every place untrusted input enters: routes/endpoints, forms,
   file uploads, query params, headers, webhooks, message queues, mobile API calls, third-party callbacks.
3. **Draw trust boundaries** — Where does data cross from less-trusted to more-trusted?
   (browser→API, API→DB, service→service, user→admin). Bugs cluster on these lines.
4. **Apply STRIDE** per boundary/component — see `references/stride-and-surface.md`.
5. **Rank** — score each threat: `risk = likelihood × impact` (1–3 each → 1–9).
   Test high scores first.

## Quick reference — STRIDE
| Threat | Question | Defends |
|--------|----------|---------|
| **S**poofing | Can someone pretend to be another user/service? | Authentication |
| **T**ampering | Can data be modified in transit/at rest? | Integrity |
| **R**epudiation | Can an action be denied / is it logged? | Non-repudiation |
| **I**nformation disclosure | Can data leak? | Confidentiality |
| **D**enial of service | Can it be made unavailable? | Availability |
| **E**levation of privilege | Can a user gain rights they shouldn't? | Authorization |

## Output: prioritized target list
A table the next phases consume:

| # | Asset | Entry point | STRIDE | Likelihood | Impact | Risk | Test via |
|---|-------|-------------|--------|-----------|--------|------|----------|
| 1 | User PII | `GET /api/users/:id` | I (IDOR) | 3 | 3 | 9 | code-audit + active-pentest |
| 2 | Auth | `POST /login` | S | 2 | 3 | 6 | code-audit |

## Hand-off
Feed the top items into **`security-code-audit`** (find) and **`active-pentest`** (validate).

**The surface inventory is also the applicability driver.** It tells the `security-testing`
orchestrator which specialists apply (surface present → run) and which to skip (absent → N/A),
and exposes any surface with **no** matching specialist as a coverage gap to handle with the
generic methodology + a recommendation to build a new skill. List every surface you found —
including ones no current skill covers — so nothing is silently missed.

## Common mistakes
- Modeling the happy path only — attackers use the *un*happy paths.
- Forgetting internal/trusted entry points (admin panels, internal APIs, cron, queues).
- Treating all findings as equal — rank, or you'll drown in low-value work.
