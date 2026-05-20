# RELEASE NOTES — Skuld v6.5.5.2

**Date:** 2026-05-19
**Branch:** polybot_skuld_v1
**Bot name:** `tps`
**Mode:** DRY
**Built on:** v6.5.5.1 (same day)

---

## What this fixes

The May 19 audit of 31 v6.5.4 orphan-sell fires found **only 7 were real** — 24 were phantom fires on stale "locked at 50/50" orderbook snapshots that the bot's websocket layer occasionally produces when price-level deltas are missed. The bot's `yes_bid`/`no_bid` froze at a phantom 0.4950/0.5050 state while the real Polymarket market had already collapsed to 0.05/0.95 or worse. Fire after fire was triggered on a price that didn't exist.

Additionally, the v6.5.4 persistence mechanism counted **consecutive shadow ticks** where all conditions held. Across 11,396 tick-pairs of held positions, the bid changed by more than 1¢ between ticks **60% of the time**. Requiring identical-condition ticks meant the rule almost never fired on legitimate profit windows — most opportunities were lost to natural price wobble.

This release addresses both root causes.

## Changes

### 1. Locked-spread reject (anti-ghost)

New helper `_bs_is_book_locked(yes_ask, no_ask, yes_bid, no_bid)` returns True when `ask == bid` on either side. Real Polymarket orderbooks always have at least 1¢ spread (the platform tick size); zero-spread is the signature of a stale in-memory book that's frozen on a phantom snapshot.

**Applied in two places:**

- **`_bs_evaluate_orphan_sell`** — first check after the early-exit guards. Locked spread → return immediately, no band-sustain updates, no fire.
- **`_bs_evaluate_bss_entry`** — defensive check after the freshness gate. Locked spread → skip this tick's entry evaluation. The book will recover on the next valid websocket update.

The book-walk simulator (`_bss_simulate_dry_sell`) is retained but still not called — kept for future LIVE realism work.

### 2. Band-based sustain (anti-wobble)

New helper `_bs_update_band_sustain(mdm, condition_met, now, first_attr, last_attr, grace_s)` replaces the tick-counting persistence with timestamp-based band tracking.

**Logic:**

- When conditions are met: set first-qualifying timestamp if None, update last-qualifying timestamp to now.
- When conditions are not met: if gap since last qualifying tick exceeds GRACE_S, reset both timestamps (sustained failure). Otherwise, leave the run intact (brief wobble is tolerated).
- The caller fires only when conditions are met NOW AND the run has lasted ≥ SUSTAIN_S.

This naturally handles wobble: a single negative tick between two positive ticks doesn't reset the run, as long as the negative window stays within GRACE_S.

**Applied to:**

- **Positive-exit rule** (orphan-sell defensive): `BS_BSS_ORPHAN_SELL_SUSTAIN_S` default **6.0s** (matches old 2-tick persist time at 3s cadence), `BS_BSS_ORPHAN_SELL_GRACE_S` default **3.0s** (wobble tolerance).
- **Take-profit rule** (orphan-sell opportunistic): `BS_BSS_ORPHAN_TP_SUSTAIN_S` default **3.0s** (matches old 1-tick persist), `BS_BSS_ORPHAN_TP_GRACE_S` default **1.0s** (TP fires fast, less wobble tolerance).

The old `BS_BSS_ORPHAN_SELL_PERSIST_TICKS` and `BS_BSS_ORPHAN_TP_PERSIST_TICKS` env vars are no longer read but remain defined for backward compatibility (won't break boot if set).

### 3. Phase visibility (analytics)

Second-leg fires were 77% "floor" phase (sustain=0s, deep-dip) and 23% "strict"/"relaxed" (timed sustain). Previously this distinction was visible only in stdout log lines, not in the BSS_LEG_FILL CSV `note` field — making it impossible to filter analytics by phase.

**`_bs_log_bss_leg_fill_event`** now accepts an optional `phase` parameter. When passed, the note includes `phase=floor` (or `phase=strict` / `phase=relaxed`). Wired up from `_bss_place_leg2`'s existing `phase_label` arg.

CSV note for leg 2 fills now looks like:
```
src=bss_entry,leg=2,phase=floor,decision_ask=0.4000,fill_ask=0.4000,…
```

Leg 1 fills are unchanged (no phase tag — they don't have one).

## What's NOT changed

- Entry sustain logic (`_bs_evaluate_bss_entry`'s `bss_yes_below_first_start_ts` / `bss_no_below_first_start_ts` streak tracking). Already band-based by design: `if ask < threshold` is a band check, not a price-point check. The `0.40 → 0.35 → 0.36 → 0.34 → 0.37` wobble already counts as one continuous sustain run — verified against the code path. No change needed.
- Polymarket curved fee math (v6.5.5).
- Cashout convention for sell pnl (v6.5.5.1) — leg1_top_bid as proxy, no book-walk gate.
- TP rule still defaults DISABLED. Flip `BS_BSS_ORPHAN_TP_ENABLED=true` to activate.
- Dashboard last-15 trades with ORPHAN_SOLD render (v6.5.5).

## Guards 1 & 2 — verified already-enforced

User raised two concerns from prior bot iterations:

- **Guard 1 — no double-entry on same market+side.** Audited May 16/17/19 data: 0 markets out of 220 had duplicate leg-1 fills. Already prevented by `_bs_evaluate_bss_entry`'s state machine — leg-1 only fires from `WATCH` state, and after fill the position transitions to `WAITING_2ND`. Additional idempotency guard at line 5595 catches edge cases (existing position → skip).
- **Guard 2 — no re-entry after orphan-sell.** Audited 31 fires: 0 re-entered. Already prevented by `ORPHAN_SOLD` being in the terminal-state list at line 5588 — the entry evaluator returns early for that state.

Both guards are intentional and tested. No new code added — but flagging here so future refactors don't accidentally break them.

## New env vars (with defaults)

```
BS_BSS_ORPHAN_SELL_SUSTAIN_S = 6.0    # band sustain time for positive-exit
BS_BSS_ORPHAN_SELL_GRACE_S   = 3.0    # wobble tolerance for positive-exit
BS_BSS_ORPHAN_TP_SUSTAIN_S   = 3.0    # band sustain time for take-profit
BS_BSS_ORPHAN_TP_GRACE_S     = 1.0    # wobble tolerance for take-profit
```

Defaults are calibrated to match prior-version behavior at the 3s shadow-tick cadence. No env changes are strictly required to deploy — the new vars all have safe defaults.

## Deprecated env vars (no action required, still defined)

```
BS_BSS_ORPHAN_SELL_PERSIST_TICKS    # no longer read (was 2)
BS_BSS_ORPHAN_TP_PERSIST_TICKS      # no longer read (was 1)
```

These remain defined to avoid boot warnings on Railway. They no longer affect bot behavior. Can be removed from Railway env at your convenience.

## Expected behavior change

Compared to v6.5.4/v6.5.5/v6.5.5.1:

- **Phantom fires:** should drop from ~77% of total to near-zero. The locked-spread filter rejects the exact signature.
- **Wobble misses:** should recover significantly. Profit windows that previously failed the 2-consecutive-tick test will now fire if conditions were met for any 6-second span (with up to 3s grace gaps).
- **DRY P&L accuracy:** dashboard P&L will reflect roughly the real LIVE-feasible captures, not the inflated phantom-included sum.

Net effect on May 19 data, retrospectively (rough estimate):
- v6.5.4 reported: +$12.97 (31 fires, 24 phantoms → only ~$1.08 real)
- v6.5.5.2 would have: rejected the 24 phantoms via locked-spread, would have additionally captured more wobble-window opportunities via band-sustain
- Hard to project exactly without running the simulator on the new logic, but expect more fires with lower fake-profit inflation and better LIVE realism

## Iron rule compliance

- **#7 fair feasibility**: Conservative — locked-spread reject can only filter MORE ticks (defensive), band-sustain can only fire MORE often within real (non-locked) windows. No new ways to lose money. Worst case: rule fires less often than before. Best case: cleaner P&L and more wobble captures.
- **#8 self-audit**: AST + compile PASS (10,362 lines). All 6 feature-presence checks PASS. All 4 old tick-counting lines verified removed from the live path. Phase wiring verified end-to-end (call sites → place_leg2 → logger → CSV note).
- **No ship without approval**: User said "go - with your recommendations - write main and whatever is necessary plus variables if any changes to current".

## Files

- `/mnt/user-data/outputs/main.py` (10,362 lines, 540 KB)
- `/mnt/user-data/outputs/RELEASE_NOTES_v6_5_5_2_skuld.md` (this file)

Existing infra files (Procfile, requirements.txt, etc.) unchanged from v6.5.5.1 — push only main.py.

## Rollout

1. Push `main.py` to Railway
2. Verify boot banner shows `v6.5.5.2 LOCKED-SPREAD REJECT` and `v6.5.5.2 BAND-SUSTAIN`
3. Watch for `BSS_ORPHAN_SELL_DRY` events over next 1-3 hours
4. Cross-check fire rate against May 19 baseline (31 fires / 9h ≈ 3.4/hour)
5. Expect: fewer total fires (no phantoms), but each fire should pass the depth-log cross-check
6. Once 24h of clean data accumulated, consider flipping TP rule on

## Outstanding (not in this release)

- TP rule still gated by env (`BS_BSS_ORPHAN_TP_ENABLED=false`) — flip when ready
- Shadow logger CSV escaping bug (cosmetic, doesn't affect bot)
- LIVE cashout API research (deferred — past LIVE used FOK→GTC fallback, will continue to)
- Wallet sync for LIVE (deferred until LIVE phase begins)
- Chainlink stream log integration (`chainlink_stream_log.py` missing from Skuld — copy from polybot_simple_v1)
- Latency tester integration (`latency_tester.py` standalone, not yet wired)

These remain on the backlog from prior sessions. No new backlog items added this release.
