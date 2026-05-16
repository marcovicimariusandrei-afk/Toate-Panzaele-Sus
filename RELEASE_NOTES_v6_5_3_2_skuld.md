# polybot_skuld_v1 — v6.5.3.2 "Skuld" — Speranța hero dashboard

**Date:** 2026-05-15
**Base:** v6.5.3.1 (hold-shadow raw-state logging)
**Output:** `main.py` (~9,525 lines), `chainlink_stream_log.py` (unchanged)
**Strategy:** unchanged — purely cosmetic dashboard change

Visual-only release. The dashboard header now renders with the Speranța
ship illustration as background, full image visible, dark glass overlay
(35%) for text legibility, with the bot title and the tagline "Toate
Pânzele Sus" centered on the hero strip.

No functional changes. No env vars added. No CSV schema changes. No
decision-logic changes. Strategy, filters, logging — all identical to
v6.5.3.1.

---

## What changes

### In source

1. **New constant `_SPERANTA_DATA_URI`** — 480×320 JPEG quality 78,
   embedded as base64 (~37KB). Self-contained; no external dependencies.

2. **New CSS rule `header.skuld-hero`** in DASHBOARD_HTML:
   - Ship image as background-image, cover fit, position center-55%
   - 130px minimum height, 24px padding
   - `::before` pseudo-element provides 35% black overlay for text contrast
   - Heading and badges get `text-shadow: 0 1px 8px rgba(0,0,0,0.7)` for legibility
   - Version badge accent shifted to amber (#f0c080) to harmonize with sunset palette

3. **HTML header element** now uses `class="skuld-hero"` and adds a
   `.skuld-tagline` span containing "Toate Pânzele Sus" in italic serif

4. **Serve-time substitution** in `_Handler.do_GET`: the placeholder
   `{{SPERANTA_BG}}` in DASHBOARD_HTML's CSS is replaced with the data URI
   on each request. ~80KB total HTML response.

5. **Boot banner** mentions `v6.5.3.2 dashboard: Speranța hero header —
   Toate Pânzele Sus`

### What stays untouched

- All v6.5.3.1 hold-shadow features
- All v6.5.3 Tier 1 logging
- v6.5.2 TTR=240 entry filter
- All trading logic, env vars, event types, CSV schemas
- Verification bot theme (`body.theme-verification`) preserved untouched

---

## Audit summary

```
✓ AST parse OK
✓ Compiles to bytecode
✓ BOT_VERSION = 6.5.3.2
✓ Header version = v6.5.3.2
✓ _SPERANTA_DATA_URI constant present (37,664 base64 chars / ~27KB raw)
✓ CSS hero rule + overlay pseudo-element + h1 styling
✓ Placeholder {{SPERANTA_BG}} in CSS, substituted at serve time
✓ Header HTML uses skuld-hero class
✓ "Toate Pânzele Sus" tagline rendered as italic serif
✓ Banner mentions Speranța hero
✓ Substitution simulation works: dashboard 43KB → 81KB rendered

No regression:
✓ _bs_log_bss_hold_shadow_event intact
✓ _v653_compute_features intact
✓ _BS_BSS_MIN_TTR_AT_LEG1_S intact
✓ _BS_BSS_SHADOW_TICK_INTERVAL_S intact
✓ chainlink_stream_log intact

Line count: 9,525 (vs v6.5.3.1 at 9,499; +26 lines actual logic + 37KB base64 data)
```

---

## Deploy checklist

1. Push `main.py` + `chainlink_stream_log.py` to GitHub Skuld repo
2. Railway auto-deploys
3. Verify boot banner: `DRY MODE v6.5.3.2 ... Speranța hero header`
4. Open dashboard URL in browser. Should show:
   - Speranța ship illustration as background of the top strip
   - Dark glass overlay making text readable
   - "polybot simple v1" title in white with shadow
   - "Toate Pânzele Sus" tagline in italic amber serif
   - Version badge with amber tint
5. All other dashboard panels (cards, BSS positions, logs) unchanged
6. `/api/status` returns `bot_version: "6.5.3.2"`

---

## Rollback

`git revert <commit>` returns to v6.5.3.1. The base64 data URI constant
becomes dead code, harmless.

Soft rollback (keep code, hide image): replace the placeholder substitution
with empty string by setting `_SPERANTA_DATA_URI = ""`. Header still
renders, just without the background image.

No env vars to unset, no schema changes to reverse.
