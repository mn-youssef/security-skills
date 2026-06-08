---
name: business-logic-testing
description: Use when testing application-specific workflow and business-rule flaws that scanners cannot find — price/quantity/parameter tampering, workflow and state-machine bypass, race conditions and TOCTOU, replay, and abuse of coupons/refunds/limits. Requires understanding what the app is supposed to do.
---

# Business Logic Testing (FIND/EXPLOIT — the pro's bread and butter)

## Overview
Find flaws in *what the app allows*, not *how it's coded*. These are valid requests that
produce invalid outcomes — buying for $0, refunding twice, skipping a required step. No scanner
finds them because each is unique to your business rules.

**Core principle:** The app enforces the *intended* flow; you test every *unintended* path that
still "works." Think: "what does this feature assume, and what if I break that assumption?"

## Applies when / Skip when
- **Applies when:** the app has multi-step workflows, money/credits/quotas, stateful processes,
  or business rules that can be abused.
- **Skip when:** it's a stateless single-action utility with no rules or state → N/A (rare —
  most apps have *some* logic; double-check before skipping).
- **If N/A:** report "business-logic-testing: N/A — no stateful workflow/rules" and stop.

## ⚠️ Authorization
Your own app/test data. Logic abuse can move money/state — use a non-production environment.

## Method
1. **Understand the intended flow** — map the steps, the rules, and the assumptions at each step
   (order: add → pay → ship; rule: qty ≥ 1; assumption: price comes from the server).
2. **Challenge each assumption** — negative/zero/huge values, skipped steps, reordered steps,
   client-supplied values the server should own, repeated actions.
3. **Attack the state machine** — reach states out of order or revisit terminal states.
4. **Attack concurrency** — do the "once only" action many times in parallel (race/TOCTOU).
5. **Attack quantity & money** — tamper price, totals, currency, discounts, rounding, limits.

See `references/logic-attack-patterns.md` for a catalog of concrete patterns.

## Quick reference — classic logic flaws
| Pattern | Example |
|---------|---------|
| Parameter tampering | Change `price`, `total`, `qty`, `userId`, `status` in the request the server trusts |
| Negative / overflow | `qty=-1` credits you; huge values overflow; `0` bypasses a charge |
| Workflow bypass | Jump straight to "confirm"/"download" without "pay"/"verify" |
| State machine abuse | Cancel an already-shipped order; reopen a closed ticket to gain rights |
| Race condition / TOCTOU | Redeem one gift card / coupon / withdrawal N times in parallel |
| Replay | Resubmit a signed/one-time request (payment callback, OTP, idempotency key) |
| Discount/refund abuse | Stack coupons, refund more than paid, partial-refund loops |
| Limit bypass | Exceed per-user caps via parallel requests or multiple sessions |

## The race-condition test (high value, often missed)
Send the "limited" action many times **simultaneously** (not sequentially) and check if it
succeeds more than once — e.g. 20 parallel redemptions of a single-use coupon. Tools: Burp
Turbo Intruder, or a quick parallel `curl`/script firing identical requests at once.

## Output
Findings describing the **intended rule**, the **request that broke it**, and the **business
impact** (money lost, limits bypassed, privilege gained). Severity = real-world impact, often Critical.

## Hand-off
Reproduce safely on staging via **`active-pentest`**; chain with auth/access-control in
**`vulnerability-chaining`**; fix via **`security-hardening`** (server-authoritative values,
atomic transactions/locks, idempotency keys, server-side state validation).

## Common mistakes
- Looking for code smells — logic bugs are in *valid* requests; you must know the *rules*.
- Testing actions sequentially when the bug only appears under **concurrency**.
- Trusting client-sent prices/totals/limits — re-derive everything server-side and test that it does.
- Ignoring "soft" features: coupons, referrals, loyalty points, trials, refunds — prime abuse targets.
