# Production Scan: Pre-Fix vs Post-Fix Comparison
**Date:** 2026-04-17  
**Bug fixed:** Learner-to-scorer key mismatch — multipliers were silently defaulting to 1.0 on every scan since the nightly learner was activated tonight.

---

## What Changed (Summary)

The nightly learner ran at ~20:52 tonight and wrote real multipliers to `data/v2_scoring_adjustments.json`. Those multipliers were never read by the scoring engine because of a three-way schema mismatch:

| Problem | Writer (run_bot.py) | Reader (v2_research_engine.py) |
|---------|--------------------|---------------------------------|
| Structural | Flat dict `{component_name: {multiplier: …}}` | Reads `data.get("multipliers", {})` → always `{}` |
| Name | `has_catalyst`, `float_shares`, `short_pct` | Expects `catalyst`, `low_float`, `high_short` |
| Value type | Stored dicts per factor | Expected plain floats |

**Fix:** Adjusted the writer (`run_nightly_learning()` in `run_bot.py`) to produce the canonical schema that the reader already expected. Chose to fix the writer because `v2_scoring_adjustments_TEST.json` confirmed the reader's format was the intended canonical schema. Also backfilled tonight's live adjustments file into the corrected format so the fix takes effect at Monday open.

**Multipliers now active:**
```
catalyst    = 0.95  (−5%  — learner found catalyst signal slightly over-weighted vs outcomes)
low_float   = 0.80  (−20% — learner found float factor significantly over-weighted)
high_short  = 0.9875 (−1.25% — minor reduction in short interest weighting)
```

---

## Before-Fix Scan (23:12 tonight — all multipliers silently 1.0)

| Rank | Symbol | Score (norm) | Raw | Tier | Key factors |
|------|--------|-------------|-----|------|-------------|
| 1 | FRMM | 76 | 55 | S | Low float 14.1M, SI 17.2%, vol 54.9x, +81.0% breakout |
| 2 | BIRD | 71 | 47 | S | Very low float 5.7M, SI 19.9%, vol 376.3x |
| 3 | ACHV | 46 | 70 | B | Biotech, float 47.1M, SI 15.6%, vol 5.6x |
| 4 | GRCE | 43 | 65 | B | Biotech, float 9.8M, SI 12.6% |
| 5 | ELTX | 36 | 55 | B | Biotech, float 12.7M, SI 10.3% |

**Multipliers confirmed as:** all 1.0 (bug active — `data.get("multipliers", {})` returned `{}`)

---

## After-Fix Scan (23:32 tonight — real multipliers loaded)

**Multipliers confirmed in logs:**
```
Loaded v2 scoring adjustments (v2): catalyst=0.95, low_float=0.80, high_short=0.99
```
(logged once per candidate, every candidate — propagation confirmed)

| Rank | Symbol | Score (norm) | Raw | Tier | Δ score | Δ tier |
|------|--------|-------------|-----|------|---------|--------|
| 1 | BIRD | 66 | 40 | S | −5 (norm), −7 (raw) | no change |
| 2 | FRMM | 60 | 30 | S | **−16 (norm), −25 (raw)** | no change |
| 3 | ACHV | 44 | 66 | B | −2 (norm), −4 (raw) | no change |
| 4 | GRCE | 38 | 57 | B | −5 (norm), −8 (raw) | no change |
| 5 | ELTX | — | — | — | dropped out of top picks |

**No tier changes** in this batch of 30 evaluated candidates. The multipliers reduced scores across the board but no candidate crossed a tier boundary in this sample. The largest impact was on FRMM (−16 pts normalized) driven by the 20% reduction in low_float weighting combined with a stale "+81% breakout" bonus expiring between scans.

**Watchlist DB top ready (cumulative, all scans):**
```
CYCN  80.0  (seen 6x)  — Biotech, float 3.5M, vol explosion
BIRD  66.0  (seen 23x) — Float 5.7M, SI 19.9%, vol 376x
FRMM  60.0  (seen 5x)  — Float 14.1M, SI 17.2%, vol 54.9x
AIXI  46.0  (seen 7x)  — Float 25.5M, SI 27.0%
PLCE  32.0  (seen 22x) — Float 7.7M, SI 30.3%
```

**Health:** Scan ran end-to-end with no unhandled exceptions. Alpaca API returned 401 (no credentials in env — expected in paper/demo state). Finviz returned 385 candidates (vs 457 in pre-fix scan — different Finviz page count is normal variance between runs). Reddit returned 65 trending tickers. All data sources behaved correctly.

---

## Known Issues Not Addressed in This Pass

### 1. Anti-Chase Gate Bypass When `times_seen` Is Elevated (NEEDS APPROVAL BEFORE CHANGE)

**Location:** `run_bot.py`, lines 914–939, `execute_trades()`.

**What's happening:** The anti-chase gate rejects stocks up >15% from previous close *unless* they were pre-identified. "Pre-identified" is defined as `times_seen > 1 OR on_premarket_wl`. This means any stock that appeared in even one prior scan gets a bypass — regardless of how much it has already moved.

**The FRMM example:** FRMM was up +81.8% from previous close tonight and appeared in the top picks at S-tier (score 76). It has `times_seen = 5` (seen 5 times across prior scans). Under the current logic it would receive a bypass: `sym_times_seen=5 > 1` → anti-chase gate skipped. An 81.8% move is not a "pre-identified setup" — it's a fully-extended chase target with no remaining squeeze.

**Relevant code:**
```python
if sym_times_seen <= 1 and not on_premarket_wl:
    # BLOCK: not pre-identified
    ...continue
else:
    # ALLOW: times_seen > 1 OR on premarket watchlist
    log.info(f"  ANTI-CHASE: {pick['symbol']} +{gap_from_prev:.1f}% BUT pre-identified "
             f"(seen={sym_times_seen}x, watchlist={on_premarket_wl}) — proceeding")
```

**Recommendation (not implemented — needs approval):** Add a hard ceiling above which `times_seen` no longer grants bypass, regardless of prior identification. Suggested thresholds to consider:

- **Conservative:** block if `gap_from_prev > 50%` (catches truly extended moves, allows moderate strength gaps)
- **Moderate:** block if `gap_from_prev > 30%`
- **Aggressive:** block if `gap_from_prev > 20%` (back to a tight gate)

The 30% ceiling would mean: "I'll buy a pre-identified setup up to +30% from yesterday's close, but not at +81%." This feels right — a setup that already moved 80% is a post-squeeze remnant, not a pre-squeeze entry.

**To implement:** Change lines 921-938 in `execute_trades()` to:
```python
HARD_CHASE_CEILING = 30.0  # pct — times_seen bypass has no effect above this
if sym_times_seen <= 1 and not on_premarket_wl:
    # standard anti-chase block
    ...continue
elif gap_from_prev > HARD_CHASE_CEILING:
    # even pre-identified stocks are too extended at this level
    log.info(f"  ANTI-CHASE: {pick['symbol']} +{gap_from_prev:.1f}% exceeds hard ceiling "
             f"({HARD_CHASE_CEILING}%) — blocking even with times_seen={sym_times_seen}x")
    ...continue
else:
    log.info(...)  # existing bypass logic
```

**DO NOT implement without user approval.** This changes execution policy, not data flow.

---

### 2. `composite_score` Factor from Learner Is Silently Discarded

**Location:** `run_bot.py` `_COMPONENT_TO_FACTOR` mapping (newly added in this fix).

**What's happening:** The nightly learner produces an adjustment for `composite_score` (multiplier 1.2 tonight — the learner found the composite score predictor to be effective). But `composite_score` has no corresponding factor in `v2_research_engine.py`'s `load_scoring_adjustments()` defaults, so it is placed in `_detail` only and never applied to scoring.

**Is this a problem?** Probably not — `composite_score` is a meta-signal about overall scoring accuracy, not a specific scorer factor. It doesn't have an obvious mapping to any single scorer factor. The right long-term fix would be a global score scalar, but that's a separate design decision.

**Immediate action needed:** None. Documenting for awareness.

---

### 3. Scanner Weights (`weights.json`) and Scorer Adjustments (`v2_scoring_adjustments.json`) Are Parallel Writes With No Cross-Check

**Location:** `run_nightly_learning()` writes to both `scanner/weights.json` (via `_apply_weight_changes()`) and `data/v2_scoring_adjustments.json`. These target two different scoring systems (v1 scanner vs v2 research engine) and use different factor names with no shared validation.

**Silent break potential:** If the learner drifts one system's weights far from the other's, the v1 scanner and v2 engine could diverge significantly over time with no alert. Tonight's weights.json shows `composite_score: 0.6, float_shares: 0.4` while the v2 engine has its own factor scale. There's no reconciliation check.

**Recommendation:** Low priority for now — the v2 engine is the active path. Flag for review if the v1 scanner is ever reactivated.

---

### 4. `v2_scoring_adjustments_TEST.json` Is a Dead Reference File

**Location:** `data/v2_scoring_adjustments_TEST.json`

**What it is:** A hand-crafted file showing the intended v2 schema (with `"multipliers"` wrapper and `"history"` key). It was clearly written as a reference for what the format should look like, but was never wired up as a fallback or default. It confirmed the intended format during this fix investigation.

**Action:** No code change needed. Consider adding a comment in `load_scoring_adjustments()` referencing this file as the schema spec, or delete it if it's confusing noise.

---

## Commit

```
fix(learning): reconcile learner output with scorer input — multipliers now propagate
```

Files changed:
- `run_bot.py` — fixed `run_nightly_learning()` writer to produce `{"version":2, "multipliers":{…}, "_detail":{…}, "_meta":{…}}`
- `data/v2_scoring_adjustments.json` — backfilled tonight's live data into corrected schema
- `tests/test_learner_scorer_propagation.py` — 4 smoke tests: propagation, regression (old format → all 1.0), score-level impact, live-file schema validation

All 4 smoke tests passing.
