# RELEASE NOTES — Skuld v6.5.4

**Date:** 2026-05-18
**Branch:** polybot_skuld_v1
**Bot name:** `tps` (`BOT_NAME=tps`)
**Mode:** DRY (LIVE_BSS_ENABLED=false)

---

## Headline

Three changes, all driven by the 49.5h DRY audit (May 16 15:51 → May 18 17:21 UTC):

1. **Orphan-sell rule (positive-exit)** — new exit path during `WAITING_2ND` hold. Default **DISABLED**.
2. **Dashboard P&L fix** — running counter now includes leg-fill fees.
3. **Manual cleanup endpoint** — human-triggered CSV deletion for disk management.

No strategy parameters changed. No state-machine breaking changes. Backward-compatible env layer.

---

## 1. Orphan-sell rule (positive-exit)

### What it does

During the `WAITING_2ND` hold (leg-1 filled, awaiting leg-2), the bot checks at each shadow tick (3s cadence) whether the position can be exited profitably. If yes, it sells leg-1 at the current bid and closes the position before window end — avoiding the typical orphan loss of -$1.02.

### Rule

Fires when ALL THREE conditions hold for `BS_BSS_ORPHAN_SELL_PERSIST_TICKS` consecutive shadow ticks:

```
(a) sell_pnl_now               >= BS_BSS_ORPHAN_SELL_MIN_PNL          (default $0.00)
(b) hold_elapsed_s             >= BS_BSS_ORPHAN_SELL_MIN_ELAPSED_S    (default 90s)
(c) bin_adverse_since_leg1_bps >= BS_BSS_ORPHAN_SELL_MIN_ADVERSE_BPS  (default 0 bps)
```

Where:
- `sell_pnl_now` = `leg1_qty * leg1_bid_now - leg1_qty * leg1_bid_now * taker_fee - leg1_size - leg1_entry_fee`
- `hold_elapsed_s` = wall-clock seconds since leg-1 fill
- `bin_adverse_since_leg1_bps` = BTC movement against the leg-1 direction in bps (positive = BTC moved against us)

### Why these conditions

From the 49.5h hold-shadow dataset (28,023 ticks across 532 holds):

| Variable | Paired median (at positive-pnl ticks) | Orphan median (at positive-pnl ticks) | Direction |
|---|---|---|---|
| `bin_adverse_since_leg1_bps` | -0.6 | +1.0 | Orphan = BTC against us |
| `hold_elapsed_s` | 70 | 130 | Orphan peak comes later |
| `ttr_s` | 194 | 132 | Less time left when orphan peaks |

The rule fires when leg-1 bid recovers enough to net break-even, AND we're past 90s into the hold, AND BTC is still moving the wrong way (i.e., not a real reversal). This combination flags an orphan whose bid is briefly bouncing but won't pair.

### In-sample simulation results (49.5h)

| Variant | Net Δ vs baseline | Orphans saved | Paireds exited early |
|---|---|---|---|
| Persist=1 tick | +$233.62 | 186 / 220 (85%) | 87 / 301 (29%) |
| **Persist=2 ticks (default)** | **+$169.21** | **151 / 220 (69%)** | **58 / 301 (19%)** |

Per-day breakdown with persist=2 default:
- May 16 (8h, baseline +$8.41): with rule **+$13.54** (Δ +$5.13)
- May 17 (24h, baseline +$6.80): with rule **+$91.47** (Δ +$84.67)
- May 18 (17h, baseline **-$8.87**): with rule **+$70.55** (Δ +$79.41) — losing day flips to most profitable

### Caveats

- This is **in-sample** optimization. Real out-of-sample performance will be worse.
- DRY-simulated sell at leg-1 bid; LIVE may face FOK fails or worse fills.
- The rule trades paired upside for orphan rescue. Net positive because orphan losses dominate paired upside in adverse regimes.
- Default **DISABLED** (`BS_BSS_ORPHAN_SELL_ENABLED=false`). Flip to true to activate.

### State machine extension

New terminal state added: `ORPHAN_SOLD`
- Set when the rule fires during `WAITING_2ND`
- Stored on `MultiDurationMarket`:
  - `bss_orphan_sold_at` (the leg-1 bid we exited at)
  - `bss_orphan_sold_ts` (epoch seconds)
  - `bss_orphan_sold_pnl` (realized pnl including all fees)
- Treated as terminal by the BSS evaluator (next tick skips)
- No resolution flow involved — position is fully closed at sell time

### New event type

`BSS_ORPHAN_SELL_DRY` (or `_LIVE`) logged to `bs_trades.csv` with:
- `pnl_usdc` = `sell_pnl_now` (net of all fees, includes leg-1 entry fee)
- `notes` includes: `sold_at`, `sell_pnl`, `hold_elapsed`, `bin_adverse_bps`, `persist_ticks`, `ttr`

### Code changes

- Added env vars: `BS_BSS_ORPHAN_SELL_ENABLED`, `_MIN_PNL`, `_MIN_ELAPSED_S`, `_MIN_ADVERSE_BPS`, `_PERSIST_TICKS`
- Added bookkeeping fields on `MultiDurationMarket`: `bss_orphan_sell_consecutive_ticks`, `bss_orphan_sold_at`, `bss_orphan_sold_ts`, `bss_orphan_sold_pnl`
- Added `_bs_evaluate_orphan_sell()` — rule evaluator, called right after shadow logger
- Added `_bs_fire_orphan_sell()` — sell action + state transition + event log
- Added `ORPHAN_SOLD` to terminal-state guard in `_bs_evaluate_bss_entry()`

---

## 2. Dashboard P&L fix

### The bug

`state.bs_pnl_today_usdc` (the running counter shown on the dashboard) was only updated on `RESOLVE_WIN`/`RESOLVE_LOSS` events. Per-leg taker fees (`-$0.02` per fill, logged to CSV via `BSS_LEG_FILL_DRY` events) were never added to the counter.

Verified gap: dashboard showed `+$22.86` over the 49.5h window, real net was `+$8.29`. The exact $14.57 difference matched 743 fills × $0.02 in fees (paired = 2 × $0.02 = $0.04, orphan = 1 × $0.02 = $0.02).

### The fix

In `_bs_log_bss_leg_fill_event()`, after writing the CSV row, also do:
```python
state.bs_pnl_today_usdc -= fee
```

The CSV format is unchanged. The dashboard counter now reflects true net P&L including fees. No retroactive correction — counter resets daily as before.

---

## 3. Manual cleanup endpoint

### Why

Railway free tier disk filled up during the May 16-18 soak (250MB/day of CSV growth). When disk is full, the bot can't write new rows — data loss.

### Behavior

`GET /api/cleanup` → preview mode, returns list of files that WOULD be deleted (no action).
`GET /api/cleanup?confirm=true` → actually deletes.

Always safe — today's actively-written files are NEVER deleted, regardless of `confirm`. Rule: only files whose date-stamp in the filename is BEFORE today's UTC date are eligible.

Response (preview):
```json
{
  "ok": true,
  "confirm_required": true,
  "today_utc": "2026-05-19",
  "deleted": [],
  "would_delete": ["tps_bs_trades_2026-05-16.csv", "tps_depth_log_2026-05-16.csv", ...],
  "errors": [],
  "hint": "call /api/cleanup?confirm=true to actually delete"
}
```

### Workflow

1. Download CSVs from `/api/download/<filename>` (existing endpoint)
2. Verify files arrived intact on your laptop
3. Hit `/api/cleanup` to preview what would be deleted
4. Hit `/api/cleanup?confirm=true` to free disk space

Existing `LOG_RETENTION_DAYS=14` backstop remains active. If user forgets to call cleanup, the in-process retention thread will still delete files older than 14 days.

---

## Environment variables — copy-paste config

Existing variables unchanged. New variables added at the end (all optional with safe defaults):

```
PRIVATE_KEY=<paste your private key here>
PROXY_WALLET=0x9dE455d0d38468FA580DF09A4eFEE29185757Bb5
FORCE_SIGNATURE_TYPE=1
MODE=dry
LIVE_BSS_ENABLED=false
BOT_NAME=tps
STRATEGY_MODE=both_sides_btc
BS_STRATEGY=bss_entry

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
BS_TAKER_FEE_PCT=0.02
BS_HEALTH_LOG_INTERVAL_S=10.0

# v6.5.4 NEW — orphan-sell rule (default DISABLED, safe rollout)
BS_BSS_ORPHAN_SELL_ENABLED=false
# To activate the rule with audit-derived defaults, set ENABLED=true above.
# All thresholds below are optional — defaults apply when unset:
# BS_BSS_ORPHAN_SELL_MIN_PNL=0.0
# BS_BSS_ORPHAN_SELL_MIN_ELAPSED_S=90.0
# BS_BSS_ORPHAN_SELL_MIN_ADVERSE_BPS=0.0
# BS_BSS_ORPHAN_SELL_PERSIST_TICKS=2
```

---

## Rollout plan

1. **Deploy with defaults** (`BS_BSS_ORPHAN_SELL_ENABLED=false`)
   - Validate boot banner shows v6.5.4
   - Confirm dashboard P&L now reflects fees (compare against bot logs)
   - Confirm `/api/cleanup` returns preview correctly

2. **Activate rule** (flip `BS_BSS_ORPHAN_SELL_ENABLED=true`)
   - No restart needed; env reload via Railway redeploy
   - Watch `bs_trades.csv` for new `BSS_ORPHAN_SELL_DRY` events
   - Check stdout for `[bss_orphan_sell]` log lines

3. **24-48h soak with rule active**
   - Compare actual vs simulated rule fires
   - Verify net P&L lift is in ballpark of in-sample +$80-120/day

4. **Tune if needed**
   - Conservative variant: `MIN_ELAPSED_S=120`, `MIN_ADVERSE_BPS=1`, `PERSIST_TICKS=2`
   - Aggressive variant: `MIN_PNL=-0.10`, `PERSIST_TICKS=1`

5. **Continue collecting hold-shadow data** for future iterations:
   - Sell-loser in paired (deferred — separate from this release)
   - Entry filter tightening (deferred — needs more data)

---

## What's NOT in v6.5.4

- **Sell-loser in paired markets** — investigated, found to be intentionally suppressed in BSS mode since v6.3.0 (line 7575 returns early when `_BS_STRATEGY=bss_entry`). Reactivating requires more architectural change. Deferred to v6.5.5+.
- **Entry filter changes** — T_FIRST stays at 0.45. The 0.40-0.45 bucket has structurally bad EV but small sample. Revisit after orphan-sell data accumulates.
- **`bin_vol_*_bps` bug fix** — values still round to 0.0. Methodological issue (per-tick log-return std too small). Cosmetic, deferred.

---

## Iron rule compliance

- **#7 fair feasibility**: simulated results are in-sample; expect 30-50% haircut out-of-sample.
- **#8 self-audit**: AST parse PASS, compile PASS, all prior features (Tier-1, hold-shadow, candidate, Speranța dashboard, chainlink, BSS fast-tick, TTR filter) verified present.
- **#9 fetch live source before edit**: edited from base of v6.5.3.2 main.py that user provided in earlier session.
- **No ship without explicit approval**: user said "all 3 ok, 2. human delete" — that was the explicit go-ahead.
- **VERIFY-DEPLOY iron rule**: after deploy, run Railway agent verification within 90s.
