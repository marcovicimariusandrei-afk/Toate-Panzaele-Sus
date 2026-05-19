# RELEASE NOTES — Skuld v6.5.5

**Date:** 2026-05-19
**Branch:** polybot_skuld_v1
**Bot name:** `tps`
**Mode:** DRY (LIVE_BSS_ENABLED=false)
**Built on:** v6.5.4 (orphan-sell shipped, currently +$13/9h validated in prod)

---

## Headline

Four changes, driven by **real production data** from May 16-19 (~59h effective soak, 354 paired + 234 orphan markets):

1. **Polymarket fee formula corrected** — bot was using flat 2% taker fee; real formula is curved (~3.5-7% depending on price). This UNDER-COUNTED fees by ~$22 over 59h. True P&L: +$3/day, not +$10/day as previously thought.

2. **Take-profit (TP) rule** — sells leg-1 when bid recovers to 1.75× entry. Complementary to v6.5.4 orphan-sell. Default DISABLED.

3. **DRY-mode realism** — sell pnl now uses walked bid book (not assumed top-of-book fill). Catches partial fills and slippage.

4. **Dashboard upgrades** — last-15 trades instead of last-10, ORPHAN_SOLD positions rendered with sell price, reason, and hold duration.

---

## 1. Polymarket fee formula (THE BIG ONE)

### What was wrong

The bot used `fee = trade_value × 0.02` (flat 2%). The real Polymarket formula for crypto markets:

```
fee = shares × 0.07 × price × (1 - price)
```

This produces a **curved fee** that peaks at p=$0.50 and falls toward extremes. For a $1 trade:
- p=0.20 → fee $0.056 (5.6%) — bot was charging $0.020 (180% under)
- p=0.30 → fee $0.049 (4.9%) — bot was charging $0.020 (145% under)
- p=0.45 → fee $0.039 (3.9%) — bot was charging $0.020 (95% under)
- p=0.50 → fee $0.035 (3.5%) — bot was charging $0.020 (75% under)

### Impact on past P&L

The 59h audit showed the bot was UNDER-COUNTING fees by ~$22:

| Metric | Old (2% flat) | Correct (Polymarket) |
|---|---|---|
| Paired P&L | +$257 | +$243 |
| Orphan P&L | -$225 | -$235 |
| **Net** | **+$32** | **+$8** |
| Daily rate | +$13/day | +$3/day |

The strategy is essentially break-even with slight positive drift, NOT +$13/day.

### Source

Verified May 18 2026 against https://docs.polymarket.com/trading/fees.

### Implementation

- New helper `_polymarket_taker_fee(shares, price)` — computes correct fee
- All 12 internal fee calculation sites migrated to use the helper
- Two new env vars:
  - `BS_POLYMARKET_TAKER_FEE_RATE=0.07` (the rate constant, override if Polymarket changes it)
  - `BS_USE_POLYMARKET_FEE_FORMULA=true` (toggle — set false to fall back to flat 2% for backward-compat sim)
- Legacy `BS_TAKER_FEE_PCT=0.02` retained as fallback only

---

## 2. Take-profit (TP) rule

### What it does

While leg-1 is held in `WAITING_2ND` state, if the bid recovers to **1.75× the entry price** for **1 consecutive shadow tick** (~3s), close leg-1 at the current bid. Lock in a real profit even when leg-2 never fires.

### Rule

```
leg1_bid_now / leg1_entry_ask >= BS_BSS_ORPHAN_TP_RATIO    (default 1.75)
for BS_BSS_ORPHAN_TP_PERSIST_TICKS consecutive ticks       (default 1)
```

### Why 1.75 ratio + persist=1

Derived from 59h audit (correct fees, first-tick fire, no peak-cherrypicking):

| Ratio | Persist | P fires | O rescued | Net/day |
|---|---|---|---|---|
| 1.50 | 1 | 177/354 (50%) | 99/234 (42%) | +$47 |
| 1.60 | 1 | 138/354 (39%) | 79/234 (34%) | +$41 |
| **1.75** | **1** | **85/354 (24%)** | **66/234 (28%)** | **+$45** |
| 1.80 | 1 | 63/354 (18%) | 59/234 (25%) | +$40 |
| 2.00 | 1 | 31/354 (9%) | 34/234 (15%) | +$27 |

1.75 is the **balanced choice** — fires on only 24% of paireds (so 76% still capture full paired upside) while rescuing 28% of orphans with substantial gain.

Persist=1 chosen because price moves up/down quickly — if the bid hit 1.75× and dropped on the next tick, we'd miss the opportunity. Single-tick confirmation = fire fast.

### How it differs from v6.5.4 orphan-sell

| | v6.5.4 orphan-sell (positive-exit) | v6.5.5 take-profit |
|---|---|---|
| **Purpose** | Defensive — exit at break-even when BTC moves wrong way | Opportunistic — lock in gain when bid spikes up |
| **Trigger** | sell_pnl ≥ $0 AND elapsed ≥ 90s AND BTC adverse ≥ 0 | bid ≥ 1.75 × entry |
| **Persist** | 2 ticks (~6s) | 1 tick (~3s) |
| **When fires** | Late in hold, in adverse BTC regimes | Anytime bid recovers strongly |
| **Typical profit captured** | $0.05–$0.30 (small recovery) | $0.30–$0.80 (substantial gain) |
| **Mutual exclusivity** | Either rule fires first wins. They don't both fire on the same hold. |

### Default DISABLED

`BS_BSS_ORPHAN_TP_ENABLED=false` by default. After verifying v6.5.5 boot + dashboard + fee fix on Railway, flip to `true` to activate. Estimated impact: **+$25-45/day** on top of the existing orphan-sell rule.

### Code changes

- New env vars: `BS_BSS_ORPHAN_TP_ENABLED`, `_RATIO`, `_PERSIST_TICKS`
- New `MultiDurationMarket` fields: `bss_orphan_tp_consecutive_ticks`, `bss_orphan_sold_reason`
- `_bs_evaluate_orphan_sell()` now evaluates BOTH triggers (orphan-sell first, then TP)
- `_bs_fire_orphan_sell()` accepts `reason` parameter ('positive_exit' or 'take_profit')
- New event types in `bs_trades.csv`:
  - `BSS_ORPHAN_SELL_DRY` (orphan-sell rule) — already in v6.5.4
  - `BSS_ORPHAN_TP_DRY` (take-profit rule) — NEW in v6.5.5

---

## 3. DRY-mode realism: bid-walk slippage on sells

### What was naive before

The orphan-sell rule computed sell pnl assuming the entire `leg1_qty` filled at `leg1_top_bid`. This overestimates proceeds when:
- Top-of-book bid has only 5 shares of size but we're selling 10
- Bid book is thin and our sell walks down through multiple levels
- Bid disappears between decision and fill moment

### What v6.5.5 does

New function `_bss_simulate_dry_sell()` mirrors `_bss_simulate_dry_fill()` but for the exit side. Walks `bid_levels` (sorted descending) until target qty is reached. Returns:
- `avg_sell_price` (VWAP across walked levels — typically slightly worse than top-of-book)
- `qty_sold` (may be less than target on thin books)
- `sell_fee` (Polymarket formula on actual avg fill)
- `outcome` ("filled_top" / "filled_walked" / "partial" / "no_book" / "no_liquidity")

### Conservative behavior

If the bid book can't fully absorb our sell (outcome=`partial`), the rule **does NOT fire** — same logic LIVE would face. Resets persist counters and waits for better liquidity. Matches what we'd do in LIVE.

### Why this matters

The 31 production fires of orphan-sell on May 19 were computed assuming perfect top-fill. v6.5.5 will produce **slightly lower realized pnls** in DRY (closer to what LIVE will see). Better to be pessimistic now than surprised when LIVE.

---

## 4. Dashboard upgrades

### Last 10 → Last 15

Trade history widget now shows the most recent **15 trades** (was 10). Wider memory means you can see orphan-sells alongside paired resolutions without losing context.

### ORPHAN_SOLD rendering

Orphan-sold positions (from either rule) now appear in the trade history with a dedicated layout:
- **Cyan TP pill** for take-profit fires
- **Orange P-EXIT pill** for positive-exit fires
- Shows: side, entry → sell price (e.g., "NO 0.225→0.395")
- Shows: hold duration ("142s held")
- Shows: trigger reason (TP shows "1.75× hit · 1.78×"; positive-exit shows "bid recovered · adv 12bps")
- Shows: realized P&L with correct fees

### Status JSON

`bs_trade_history` in `/api/status` now includes orphan-sold entries with full sell-side fields (`sell_price`, `sell_qty`, `sell_pnl`, `sell_reason`, `hold_elapsed_s`, `bin_adverse_bps`, `tp_ratio`).

---

## Environment variables — full config

```
PRIVATE_KEY=<paste>
PROXY_WALLET=0x9dE455d0d38468FA580DF09A4eFEE29185757Bb5
FORCE_SIGNATURE_TYPE=1
MODE=dry
LIVE_BSS_ENABLED=false
BOT_NAME=tps
STRATEGY_MODE=both_sides_btc
BS_STRATEGY=bss_entry

# Entry filter and BSS knobs (unchanged from v6.5.4)
BS_BSS_T_FIRST=0.45
BS_BSS_SUSTAIN_FIRST_S=4.0
BS_BSS_T_SECOND_STRICT=0.50
BS_BSS_T_SECOND_RELAXED=0.62
BS_BSS_SUSTAIN_SECOND_S=3.0
BS_BSS_RELAX_AT_S=240.0
BS_BSS_T_SECOND_FLOOR=0.40
BS_BSS_BTC_VEL_FILTER_PCT=0.02
BS_BSS_BTC_VEL_LOOKBACK_S=30.0
BS_BSS_TICK_INTERVAL_S=0.05
BS_BSS_MIN_TTR_AT_LEG1_S=210.0
BS_BSS_SHADOW_TICK_INTERVAL_S=3.0
BS_LEAD_TIME_MIN_S=60
BS_LEAD_TIME_MAX_S=1800
SKIP_END_MINUTES=0
LOG_TO_DISK=true
LOG_RETENTION_DAYS=14
BS_BOOK_WALK_ENABLED=true
BS_HEALTH_LOG_INTERVAL_S=10.0

# v6.5.4 orphan-sell (positive-exit) — already enabled in prod
BS_BSS_ORPHAN_SELL_ENABLED=true        # ← leave enabled per current prod
# BS_BSS_ORPHAN_SELL_MIN_PNL=0.0       (default OK)
# BS_BSS_ORPHAN_SELL_MIN_ELAPSED_S=90.0
# BS_BSS_ORPHAN_SELL_MIN_ADVERSE_BPS=0.0
# BS_BSS_ORPHAN_SELL_PERSIST_TICKS=2

# v6.5.5 NEW — Polymarket fee formula (defaults match docs)
# BS_POLYMARKET_TAKER_FEE_RATE=0.07    (default 0.07 for crypto)
# BS_USE_POLYMARKET_FEE_FORMULA=true   (default true — set false ONLY to revert)

# v6.5.5 NEW — take-profit rule (default DISABLED, flip after verification)
BS_BSS_ORPHAN_TP_ENABLED=false         # ← flip to true to activate TP
# BS_BSS_ORPHAN_TP_RATIO=1.75          (default OK)
# BS_BSS_ORPHAN_TP_PERSIST_TICKS=1     (default fires fast)

# Legacy fallback (only used if BS_USE_POLYMARKET_FEE_FORMULA=false)
# BS_TAKER_FEE_PCT=0.02
```

---

## Rollout

1. **Deploy v6.5.5 with TP DISABLED.** Confirm boot banner shows:
   ```
   *** DRY MODE v6.5.5 ... v6.5.4 orphan-sell: ENABLED ... 
   v6.5.5 take-profit (TP): DISABLED ... 
   v6.5.5 fee formula: corrected to Polymarket curved ...
   v6.5.5 DRY sells: bid-walk slippage ...
   v6.5.5 dashboard: last-15 trades with ORPHAN_SOLD render ***
   ```

2. **Verify fee fix is live.** Check `/api/status` returns `taker_fee_pct: 0.02` (legacy field; informational only). Dashboard P&L counter should track slightly lower than v6.5.4 due to higher real fees.

3. **Watch existing orphan-sell behavior.** Fewer marginal fires expected (the +$0.0008 borderline ones from v6.5.4 won't trigger anymore — correct behavior). Average fire pnl should go UP because only stronger recoveries qualify.

4. **Flip TP on.** Set `BS_BSS_ORPHAN_TP_ENABLED=true`. Watch for `BSS_ORPHAN_TP_DRY` events. Expected: 8-15 fires/day with avg pnl +$0.40-0.60.

5. **24h soak.** Compare actual TP behavior vs simulation expectation (+$25-45/day).

6. **Continue collecting data** for future iterations:
   - Sell-loser in paired (deferred — needs separate work)
   - Entry filter tightening (deferred — needs more data)

---

## Iron rule compliance

- **#7 fair feasibility**: Simulation predicted +$113/day for orphan-sell in-sample; real production fired 31× / 9h = ~+$35/day. Expect TP rule to similarly haircut from +$45/day → +$20-30/day realistic.
- **#8 self-audit**: AST parse PASS, compile PASS, feature presence audit (15/15 PASS, 1 false positive on quote pattern). All prior v6.5.0 → v6.5.4 features verified present.
- **#9 fetch live source**: Built on top of /home/claude/main.py v6.5.4 (already validated this session).
- **No ship without approval**: User said "lets go for the middle" (= 1.75 ratio) and "yes" to lowering persist (= 1 tick) and "dry mode has to be realistic" (= bid-walk slippage).
- **Verify-deploy iron rule**: Run Railway agent verification within 90s of push.
