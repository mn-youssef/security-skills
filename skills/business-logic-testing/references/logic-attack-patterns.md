# Business Logic Attack Patterns

> Non-production, your own app. These move state/money — test deliberately.

## Map first, then attack
For each feature write down: **steps** (the flow), **rules** (qty ≥ 1, one coupon per order),
**who owns each value** (price should be server-owned). Every assumption is a test.

## Parameter tampering (server trusts client values it shouldn't)
- **Price/total:** intercept checkout, change `price`/`amount`/`total` to `0`/`0.01`/negative.
- **Identity:** change `userId`/`accountId`/`from` to act as/charge someone else.
- **Status/role:** set `status:"paid"`, `isApproved:true`, `role:"admin"` in the body.
- **Currency:** pay in a weaker currency while priced in a stronger one; rounding abuse.
- **Hidden fields:** modify values the UI marks read-only/disabled.

## Numeric edge cases
- Negative quantities → refunds/credits (`qty=-5` adds balance).
- Zero → free or division issues.
- Very large → integer overflow, memory/cost blowups.
- Decimals where integers expected (`qty=1.9999`), scientific notation (`1e3`).

## Workflow / sequence bypass
- Skip steps: call the final endpoint (`/confirm`, `/download`, `/ship`) without the prerequisite
  (`/pay`, `/verify`, `/kyc`).
- Reorder steps: do step 3 before step 2.
- Reuse a step's artifact in a different context (a token issued for action X used for Y).
- Force-complete: set the "completed" flag yourself.

## State-machine abuse
- Act on terminal states: cancel a shipped order (refund + keep goods), edit a submitted exam.
- Revert states to regain privileges: reopen a closed/escalated ticket; un-expire a trial.
- Double transitions: approve→approve, redeem→redeem.

## Race conditions / TOCTOU (concurrency)
Fire identical requests **simultaneously** at "check-then-act" operations:
- Single-use coupon / gift card redeemed N times.
- Withdraw/transfer more than the balance (balance read before each parallel debit).
- Exceed per-user limits (one signup bonus claimed many times).
- Inventory oversell; vote/like multiple times.
```bash
# crude parallel fire (use Burp Turbo Intruder for true single-packet timing)
for i in $(seq 1 20); do
  curl -s -X POST URL/api/coupon/redeem -H "Authorization: Bearer $TOKEN" \
       -d '{"code":"ONCE"}' & done; wait
```
Fix: atomic DB transactions, row locks (`SELECT ... FOR UPDATE`), unique constraints, idempotency keys.

## Replay & idempotency
- Resubmit one-time requests: payment webhooks/callbacks, OTP submissions, "claim reward".
- Reuse idempotency keys/nonces — are they enforced once?
- Replay signed requests whose signature doesn't cover a timestamp/nonce.

## Discount / refund / loyalty abuse
- Stack or reuse coupons; apply expired/other-user coupons.
- Refund > paid; refund then cancel-refund loop; partial refunds summing over total.
- Earn loyalty/referral points then reverse the qualifying action while keeping points.
- Trial abuse: reset trials via re-registration / parameter change.

## Quantity & limit bypass
- Per-user caps enforced per-session only → use parallel sessions.
- Rate/spend limits checked client-side or per-endpoint but not on the bulk/import path.

## Impact framing (for the report)
State it in business terms: "An authenticated user can purchase any item for $0 by editing the
`price` field — direct revenue loss, unauthenticated-equivalent fraud." That's what gets fixed.
