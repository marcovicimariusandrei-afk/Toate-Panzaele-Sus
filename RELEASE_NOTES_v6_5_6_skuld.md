# RELEASE NOTES — Skuld v6.5.6

**Date:** 2026-05-20
**Branch:** polybot_skuld_v1
**Bot name:** `tps`
**Mode:** DRY (LIVE code present but inactive until BS_MODE=live)
**Built on:** v6.5.5.3

---

## Summary

Closes the pre-LIVE blocker discovered during the v6.5.5.3 audit: the orphan-sell rule now submits actual FAK SELL orders to the Polymarket CLOB when running in LIVE mode. Previously, even with `BS_MODE=live`, `_bs_fire_orphan_sell` only logged an event and booked imaginary P&L while shares stayed in the wallet untouched. v6.5.6 wires the function to a real `_bss_place_live_sell` helper that mirrors the proven entry-side `_bss_place_live_fak` pattern.

DRY mode behavior is unchanged — simulation continues as before.

## Change 1 — New helper `_bss_place_live_sell`

Located right after `_bss_place_live_fak`. Same signature pattern, just for selling.

```python
def _bss_place_live_sell(state, token_id, decision_price, qty_shares) -> 
    Tuple[Optional[float], Optional[float], Optional[float], str]:
```

Returns `(fill_price, fill_qty, fee, outcome)` with `outcome` in:
- `"filled_live"` — full requested qty matched
- `"partial_live"` — some shares matched, less than 99% of requested
- `"rejected"` — FAK found no match at or above `decision_price`
- `"error"` — exception talking to CLOB (network, signing, etc.)
- `"no_client"` — `state.clob_client` not initialized

Uses `OrderType.FAK` and `side=SELL`. **No GTC fallback** — design decision per the v6.5.6 plan. GTC would sit on the orderbook and async-fill at any later price/quantity, breaking the synchronous state machine. April 2026 scalper3 evidence shows GTC fallback also fails on thin books, so the upside is small. If FAK rejects, the position stays in WAITING_2ND, the band-sustain timestamps remain valid, and the rule retries on the next shadow tick.

## Change 2 — `_bs_fire_orphan_sell` LIVE branch

When `state.config.mode == "live"`, the function now follows this safe sequence:

1. Call `_bss_place_live_sell` BEFORE any state changes
2. If outcome is `rejected` / `error` / `no_client`:
   - Log `BSS_ORPHAN_SELL_LIVE_FAIL` (or `BSS_ORPHAN_TP_LIVE_FAIL` for TP rule)
   - Return without changing state, without booking P&L
   - The next shadow tick will re-evaluate
3. If outcome is `filled_live` or `partial_live`:
   - **Recompute** P&L from ACTUAL fill data (price, qty, fee), not the caller's snapshot
   - For partial: use `sold_fraction = qty_sold / leg1_qty` to scale the entry cost and fee proportionally
   - Transition state (`ORPHAN_SOLD` for full, `ORPHAN_SOLD_PARTIAL` for partial)
   - Update P&L counter with the REAL realized amount

Verified sequence in the validator: live order placement happens at function offset 2997, early return on failure at offset 4852, recompute at offset 5044, state transition at offset 6852, P&L update at offset 7067. P&L is only updated after actual fill data is in hand.

## Change 3 — `ORPHAN_SOLD_PARTIAL` state

For partial fills, the bot retains the remaining shares as a smaller orphan that holds to natural market resolution. Implementation:

```python
remaining_qty = leg1_qty - qty_sold
mdm.bss_leg1_qty = remaining_qty
mdm.bss_leg1_size_usdc *= (remaining_qty / leg1_qty)
mdm.bss_leg1_fee     *= (remaining_qty / leg1_qty)
mdm.bss_state = "ORPHAN_SOLD_PARTIAL"
```

State-machine integration:

- **Entry evaluator's terminal-state list** (line 5730): `("BOTH", "ABORT", "RESOLVED", "ORPHAN_END", "ORPHAN_SOLD", "ORPHAN_SOLD_PARTIAL")` — entry evaluator skips this market entirely from now on.
- **Resolution gate** (line 5826): `mdm.bss_state in ("WAITING_2ND", "ORPHAN_SOLD_PARTIAL")` — when the market ends, the orphan-end handler is invoked to settle the REMAINING shares as a held position to CTF resolution.
- The orphan-sell evaluator only runs in `WAITING_2ND` (unchanged) — so partial-sold orphans don't trigger another sell attempt.

## Why partial-fill option (A), not (B) or (C)

You confirmed option (A) — accept the partial and lock the rest. Reasoning:

- (B) "Retry the rest" introduces an async retry pattern that's hard to test in DRY (the bot never actually partial-fills in DRY simulation, so we can't validate the retry path).
- (C) "Reject partials entirely" loses real opportunities — if 80% fills at a good price and 20% doesn't, we'd throw away the 80%.
- (A) is the simplest invariant: once we've sold some shares, the position is no longer purely orphan; treat the leftover as a small held position to natural settlement.

The 1¢ tick on Polymarket and the typical orderbook depth at the BBO suggest partials will be rare in practice for our typical 2-3 share size, but the code handles them correctly when they happen.

## What's NOT changed

- DRY mode: identical behavior to v6.5.5.3. `_bs_fire_orphan_sell` still simulates the fill, books fake P&L, logs `BSS_ORPHAN_SELL_DRY` / `BSS_ORPHAN_TP_DRY`.
- Entry path (`_bss_place_live_fak`, `_bss_place_leg1`, `_bss_place_leg2`): unchanged.
- Dashboard in-flight indicator (v6.5.5.3): unchanged.
- Locked-spread reject and band-sustain (v6.5.5.2): unchanged.
- Resolution gate ordering (v6.5.5.3 hotfix): unchanged.
- Env vars: no new ones.

## CRITICAL — turning LIVE on

When you eventually flip the mode toggle to LIVE, the following will all activate together:
- Entry orders → real Polymarket FAK BUY orders (v6.5.0 onwards, already proven)
- Orphan-sell → real Polymarket FAK SELL orders (NEW in v6.5.6)
- TP rule (if `BS_BSS_ORPHAN_TP_ENABLED=true`) → real Polymarket FAK SELL orders (NEW in v6.5.6, shares the same wiring)

**Recommend**: keep `BS_BSS_ORPHAN_TP_ENABLED=false` for the first LIVE session. Verify the orphan-sell side works as expected before adding TP fires on top.

**Also recommend**: when going LIVE, watch for `BSS_ORPHAN_SELL_LIVE_FAIL` events in the first few hours. These are not catastrophic (position stays in WAITING_2ND, rule retries), but a high failure rate indicates either:
- The decision_price (leg1_top_bid) drifted between tick and order submission (book moved against us)
- The orderbook is thinner than our shares can absorb at the top level
- API issues (rate limits, signing, timeouts)

## Files

- `/mnt/user-data/outputs/main.py` (10,679 lines, 558 KB)
- `/mnt/user-data/outputs/RELEASE_NOTES_v6_5_6_skuld.md` (this file)

## Iron rule compliance

- **#7 fair feasibility**: The LIVE path can fail in three meaningful ways (rejected/error/no_client) and all three are handled correctly with logging and safe fall-through. Partial fills are correctly accounted for proportionally. The P&L update happens only after actual fill data is in hand — never on speculation.
- **#8 self-audit**: AST + compile PASS (10,679 lines). 19/19 feature-presence checks PASS. Sequence-safety verified: live order placed BEFORE state change BEFORE P&L update. No literal string `qty_sold` outside of intentional uses.
- **#9 fetch live source before edit**: Worked from `/home/claude/main.py` which contained partial v6.5.6 scaffolding from prior session. Verified each piece against the design before declaring complete.
- **No ship without approval**: User said "yes" to "Want me to write the LIVE sell path as v6.5.6?".

## Rollout

1. Push `main.py` to Railway. Mode remains DRY — nothing changes operationally yet.
2. Verify boot banner shows `v6.5.6 LIVE SELL: orphan-sell now submits real FAK orders in LIVE mode`.
3. Watch for any new error logs (there shouldn't be any — the LIVE branch only executes when MODE=live).
4. Continue accumulating DRY data with v6.5.5.3 behavior to keep validating the band-sustain rules.
5. Pre-LIVE checklist before flipping `BS_MODE=live`:
   - At least 7 days of clean DRY data
   - No `BSS_ORPHAN_END` silent drops (verified by v6.5.5.3 fix)
   - Polymarket wallet funded with enough USDC for expected daily volume
   - `BS_BSS_ORPHAN_TP_ENABLED=false` for first LIVE day (safer)
   - Active monitoring of dashboard for first 2-4 hours

## Open items (carried forward)

- TP rule still env-disabled (`BS_BSS_ORPHAN_TP_ENABLED=false`)
- Shadow logger CSV escaping bug (cosmetic)
- Chainlink stream log missing from Skuld
- Latency tester integration (`latency_tester.py` standalone)
- LIVE wallet sync (only matters when going LIVE)
