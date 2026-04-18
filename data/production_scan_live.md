# Production Scan — Live Run
**Run date:** 2026-04-17 (Friday, after-hours)  
**Run time:** 23:10:59 – 23:12:17 PT (78 seconds)  
**Interpreter:** `/usr/local/bin/python3` — Python 3.13.5 (same interpreter the launchd daemon uses)  
**Mode:** Dry run — no orders submitted. All source calls are real. All data is live from actual endpoints.

---

## Source Summary

| Source | Result | Notes |
|--------|--------|-------|
| Finviz | ✅ 360 tickers | All 4 scans returned data |
| Alpaca movers | ✅ 92 tickers | Weekend data — volume = 0 for all (expected) |
| Reddit | ✅ 65 trending tickers | 169 posts scanned across 5 subreddits |
| **Total unique** | **457 candidates** | After dedup across sources |

---

## Source Detail

### Finviz (360 tickers)

Four screener queries ran sequentially with 1s sleep between each.

| Scan | Filters | Result |
|------|---------|--------|
| Scan 1 — Short Squeeze | MCap ≥ micro ($50M+), float short >10%, RelVol >2x, Price <$20 | **56** |
| Scan 2 — Unusual Volume | MCap ≥ small ($300M+), RelVol >5x, Price <$20, Up | **14** (18 raw, 4 were dups) |
| Scan 3 — Biotech | Industry=Biotechnology, MCap ≥ micro, RelVol >2x, Up | **22** |
| Scan 4 — Pre-Catalyst | Float short >15%, MCap ≥ micro, Price <$20, NO volume req | **268** |

Scan 4 is the volume-monster: 16 pages of results. This is the "catch setups before they move" scan — it pulled 268 stocks with just short interest + price requirements, no volume filter. It's also why the deep-dive pool is dominated by Finviz candidates.

### Alpaca Gainers (47 unique)

Weekend behavior: all volume and trade_count fields returned as 0. Percent changes are stale (last trading session). The gainer list is real Friday data, not garbage — it's just that no new volume accumulated today (Saturday).

**Top 10 gainers from Friday's session:**
| Symbol | % Change |
|--------|----------|
| EFOI | +210.5% |
| ORIQW | +120.5% |
| FRMM | +81.0% |
| CRMX | +70.7% |
| CRMU | +69.7% |
| SVREW | +67.7% |
| BZAIW | +56.4% |
| BBGI | +53.6% |
| NOEMR | +49.0% |
| GFAIW | +48.8% |

Note: `ORIQW`, `CRMX`, `CRMU`, `SVREW`, `BZAIW`, `NOEMR`, `GFAIW` are warrants or long tickers (5-char with W/suffix) — they pass the current filter (`len(sym) <= 5`). Only plain equity tickers make it into the research DB.

### Alpaca Most-Actives (45 unique after dedup with gainers)

By volume, Friday session. Weekend volume = 0 but trade counts are real.

**Top 10 most-active (by volume):**
| Symbol | Volume (shares) | Trade Count |
|--------|----------------|-------------|
| ISPC | 600,099,393 | 407,318 |
| BMNU | 206,672,965 | 77,719 |
| BYND | 197,907,381 | 209,595 |
| TZA | 167,026,737 | 49,335 |
| NVDA | 160,946,026 | 1,917,927 |
| TSLL | 157,276,738 | 351,913 |
| ZSPC | 133,815,671 | 101,541 |
| BITO | 131,007,103 | 60,966 |
| NFLX | 126,108,536 | 1,636,650 |
| INTC | 118,838,551 | 709,018 |

ISPC (#1 by volume with 600M shares) is the signal for Monday morning — that kind of volume on a micro/small cap is worth watching.

### Reddit (65 trending, 169 posts)

5 subreddits scanned: pennystocks, squeezeplays, shortsqueeze, wallstreetbets, stocks.

**Top 30 mentions (≥ 2 to qualify as "trending"):**
| Ticker | Mentions | Note |
|--------|----------|------|
| RDDT | 29 | Reddit itself — noise |
| NVDA | 11 | Mega-cap noise |
| HNST | 6 | — |
| AEHR | 5 | — |
| TSMC | 4 | Not US-listed ticker |
| APRIL | 4 | Likely noise (month name) |
| WHAT | 4 | Likely noise |
| BIRD | 4 | Cross-signal with Finviz ✓ |
| MLTX | 4 | — |
| DXYZ | 4 | — |
| BTIG | 4 | Financial firm acronym, likely noise |
| AVGO | 4 | Mega-cap noise |
| POET | 4 | — |
| SOC | 4 | Cross-signal with Finviz ✓ |
| CIBC | 4 | Canadian bank, not US ticker |
| MSTR | 4 | — |

Reddit cross-signals with Finviz: BIRD (4 mentions), SOC (4 mentions), BYND (3 mentions), UMAC (3 mentions), RCAT (2 mentions).

Only tickers with ≥ 3 mentions AND not already in the candidate pool get added as Reddit-only candidates (v2_research_engine.py:1035).

---

## Deep Dive Selection

The 457 candidates were sorted by `(is_finviz_source, social_mentions)` descending. **Top 30 were selected for full yfinance + Finviz detail pulls.** The other 427 were never individually researched tonight.

Deep-dive ran in ~78 seconds total. Finviz detail was fetched for the first 20 candidates only (API rate limit protection, line 1052). Candidates 21–30 received only yfinance data.

**Why these 30?** The sort heavily favors Finviz sources (True > False), then social mention count. This means Reddit-only tickers with 3–4 mentions got research time, while many Finviz scan4 pre-catalyst stocks with no social signal were still included because they came from Finviz.

---

## Scoring Adjustments Applied Tonight

**All multipliers = 1.0 (defaults).**

The learning loop ran on 2026-04-17 at 20:52 and wrote `data/v2_scoring_adjustments.json` with adjustments for `composite_score`, `float_shares`, `short_pct`, and `has_catalyst`. However, the scoring engine reads `data.get("multipliers", {})` — a key that doesn't exist in the file. The learned weights are **never applied**. This is a silent config mismatch. The engine behaved as if no learning has occurred.

---

## Per-Candidate Scoring Trace (All 30)

Format: `SYMBOL [tier] raw→normalized — key factors — VERDICT`

| # | Symbol | Raw | Norm | Tier | Key factors | Verdict |
|---|--------|-----|------|------|-------------|---------|
| 1 | BIRD | 47 | 71 | **S** | Float 5.7M, SI 19.9%, VolAccel 376x, +5d +327%, Trifecta | KEEP |
| 2 | SOC | 2 | 1 | PASS | SI 22%, but float 75M, VolAccel 0.75x, -30 no-catalyst | reject |
| 3 | BYND | 20 | 13 | PASS | SI 31%, float 447M (huge), VolAccel 3.5x, -30 no-catalyst | reject |
| 4 | UMAC | 2 | 1 | PASS | SI 15.7%, VolAccel 0.93x (below threshold), -30 no-catalyst | reject |
| 5 | EOSE | 12 | 8 | PASS | SI 29.3%, float 335M, VolAccel 2.2x, -30 no-catalyst | reject |
| 6 | RCAT | 2 | 1 | PASS | SI 31.4%, float 107M, VolAccel 0.54x, -30 no-catalyst | reject |
| 7 | SNBR | 22 | 14 | PASS | SI 28%, float 18.8M, VolAccel 1.5x, -30 no-catalyst | reject |
| 8 | ACHV | 70 | 46 | **B** | Biotech (+15), float 47M, SI 15.6%, VolAccel 5.6x | KEEP |
| 9 | ACI | -10 | 0 | PASS | SI 12%, float 351M, VolAccel 1.9x, -30 no-catalyst | reject |
| 10 | ADTN | -10 | 0 | PASS | SI 11.6%, float 79M, near 52w high, -30 no-catalyst | reject |
| 11 | ALMU | 35 | 23 | C | Float 13.4M, SI 23.7%, VolAccel 4.0x, -30 no-catalyst | no trade |
| 12 | AMC | 0 | 0 | PASS | SI 15.3%, float 525M, -30 no-catalyst | reject |
| 13 | APLE | -10 | 0 | PASS | SI 16.7%, float 218M, near 52w high, -30 no-catalyst | reject |
| 14 | ARQQ | 45 | 30 | C | Float 5.3M, SI 9.7%, VolAccel 5.5x, near 52w low | no trade |
| 15 | ARRY | 10 | 6 | PASS | SI 20.2%, float 139M, VolAccel 2.3x, -30 no-catalyst | reject |
| 16 | BARK | 30 | 20 | C | Float 5.4M, SI 14.4%, VolAccel 1.1x, -30 no-catalyst | no trade |
| 17 | BMEA | 45 | 30 | C | Biotech (+15), SI 21%, float 67M, VolAccel 1.1x | no trade |
| 18 | CADL | 45 | 30 | C | Biotech (+15), SI 24%, float 63M, VolAccel 1.8x | no trade |
| 19 | CRML | 30 | 20 | C | SI 38.7%, float 73M, VolAccel 4.7x, +35.5% today, -30 no-catalyst | no trade |
| 20 | CTLP | -10 | 0 | PASS | SI 11.6%, near 52w high, -30 no-catalyst | reject |
| 21 | ELDN | 35 | 23 | C | Biotech (+15), SI 10.3%, float 69M, VolAccel 1.9x | no trade |
| 22 | ELTX | 55 | 36 | **B** | Biotech (+15), float 12.7M, SI 10.3%, VolAccel 1.7x | KEEP |
| 23 | FIG | 0 | 0 | PASS | SI 17.5%, float 216M, -30 no-catalyst | reject |
| 24 | FLYX | 15 | 10 | PASS | Float 10.4M, VolAccel 11.7x, SI 0.02% (no squeeze), -30 no-catalyst | reject |
| 25 | FRMM | 55 | 76 | **S** | Float 14.1M, SI 17.2%, VolAccel 54.9x, +81% today, Trifecta | KEEP |
| 26 | FRSH | -25 | 0 | PASS | SI 9.1%, float 224M, -30 no-catalyst, compound pump penalty | reject |
| 27 | GEVO | -10 | 0 | PASS | SI 13%, float 226M, -30 no-catalyst | reject |
| 28 | GPRE | 0 | 0 | PASS | SI 20.2%, float 67M, VolAccel 1.2x, -30 no-catalyst | reject |
| 29 | GPRO | 15 | 10 | PASS | SI 13.5%, float 123M, VolAccel 5.7x, -30 no-catalyst | reject |
| 30 | GRCE | 65 | 43 | **B** | Biotech (+15), float 9.8M, SI 12.6%, VolAccel 1.8x | KEEP |

**S/A tiers from tonight's deep dive:** FRMM (76), BIRD (71)  
**B tiers from tonight's deep dive:** ACHV (46), GRCE (43), ELTX (36)  
**C (no trade):** ALMU, ARQQ, BARK, BMEA, CADL, CRML, ELDN  
**PASS (eliminated):** 18 candidates

---

## Rejection Reasons — Tonight's 30

### ① No Catalyst Penalty (-30 raw) — 20 candidates
Every stock without a bullish news signal or biotech sector gets -30 before normalization. With max_possible=150, this alone reduces normalized score by 20 points.

Examples: SOC, BYND, UMAC, EOSE, RCAT, SNBR, ACI, ADTN, AMC, APLE, ARRY, CRML, CTLP, FIG, FLYX, FRSH, GEVO, GPRE, GPRO, RCAT

**Rule (v2_research_engine.py:843–847):** `catalyst_score <= 0` triggers `-30 * adj["no_catalyst_penalty"]`

### ② Large Float — eliminated 10+ candidates
Float > 100M hard-limits the squeeze math. With no float score contribution and the no-catalyst penalty, these candidates typically score 0–13.

Examples: SOC (75M), BYND (447M), EOSE (335M), RCAT (107M), ACI (351M), APLE (218M), AMC (525M), ARRY (139M), FIG (216M)

**Rule (v2_research_engine.py:857–858):** "Large float limits squeeze potential" — risk logged but no additional penalty beyond lack of float score.

### ③ Compound Pump-and-Dump Penalty (-15 raw) — 1 candidate
FRSH: no catalyst + no squeeze setup (SI 9.1%, float 224M) + vol_accel 2.4x > 2. Hit both the -30 and -15, resulting in raw_score = -25.

**Rule (v2_research_engine.py:851–856):**
```python
if not has_squeeze_setup and vol_accel > 2:
    score -= 15 * adj["no_catalyst_penalty"]
```

### ④ Low Short Interest — 1 candidate
FLYX: float 10.4M (good), vol_accel 11.7x (excellent), but SI = 0.02%. No short pressure = no squeeze. No SI contribution + no-catalyst penalty = score 10.

### ⑤ Near 52-week high risk flag
ADTN, APLE, CTLP, ELTX — flagged as risks but not penalized beyond the missing "near 52w low" bonus.

---

## Persistent Watchlist State (from DB)

Five candidates were already in the DB with "ready" status from previous scans. Three had score ≥ 50 and passed `get_trade_candidates()`:

| Symbol | DB Score | Times Seen | Catalyst | Sector |
|--------|----------|------------|---------|--------|
| CYCN | 80.0 | 6 | biotech_potential | Healthcare |
| FRMM | 76.0 | 4 | none | Technology |
| BIRD | 71.0 | 22 | none | Consumer Cyclical |

CYCN was **not in tonight's deep-dive 30** — it was already in the DB from prior scans (seen 6 times). It surfaced via `get_trade_candidates()` directly. Its thesis: Biotech + 3.5M float + 2342x volume acceleration + 5 social mentions. Score 80 = S tier.

AIXI (46) and PLCE (32) are also "ready" in DB but fall below the `ai_score >= 50` cutoff in `get_trade_candidates()`, so they never reach the execution path.

---

## The Buy List — What the Bot Would Actually Do Monday

Three candidates would proceed to order execution. Anti-chase check and position sizing per `run_bot.py:867–965`:

### 1. CYCN — S Tier (score 80)
- **Price tonight:** $2.98 (gap from prev close: -0.3% — irrelevant)
- **Allocation:** $100 (S tier = MAX_PER_TICKER)
- **Qty:** 33 shares × $2.98 = $98.34
- **Stop:** -20% from entry = stop at ~$2.38
- **Trailing:** Activates at +20% ($3.58 peak), trails -10% from peak
- **Thesis:** Biotech | float 3.5M | vol_accel 2342x | seen 6x
- **Anti-chase:** gap = -0.3%, no gate triggered
- **Catalyst flag:** biotech_potential ✓

### 2. FRMM — A Tier (score 76)
- **Price tonight:** $4.30 (gap from prev close: +81.8% — ABOVE the 15% anti-chase threshold)
- **Anti-chase check:** gap 81.8% > 15% → WOULD normally skip
- **BYPASS reason:** `times_seen = 4 > 1` → pre-identified in prior scans, gate bypassed
- **Allocation:** $75 (A tier = MAX_PER_TICKER × 0.75)
- **Qty:** 17 shares × $4.30 = $73.10
- **Stop:** -15% from entry = stop at ~$3.66
- **Thesis:** Float 14.1M | SI 17.2% | vol_accel 54.9x | +81% Friday | Trifecta
- **⚠ No catalyst identified** — risk noted in output

### 3. BIRD — A Tier (score 71)
- **Price tonight:** $10.915 (gap from prev close: -0.14% — no gate triggered)
- **Allocation:** $75 (A tier)
- **Qty:** 6 shares × $10.915 = $65.49
- **Stop:** -15% from entry = stop at ~$9.28
- **Trailing:** Activates at +20% (~$13.10 peak), trails -10%
- **Thesis:** Float 5.7M | SI 19.9% | vol_accel 376x | seen 22x | Trifecta
- **⚠ No catalyst identified** — risk noted in output

**Total capital that would deploy:** $98.34 + $73.10 + $65.49 = **$236.93** of $300 daily budget

---

## What's Different Monday Morning vs Tonight

1. **Volume data will be real.** All Alpaca gainer volumes were 0 tonight (weekend). Monday morning they'll reflect live premarket volume.

2. **yfinance volume_acceleration will update.** The 376x and 2342x accelerations for BIRD and CYCN reflect Friday's close vs prior 10-day average. If those stocks stabilize over the weekend, Monday's numbers will be lower. If they continue moving premarket, higher.

3. **Finviz scan1/scan2 will reflect fresh short-interest and relative volume.** Scan4 (pre-catalyst, 268 stocks tonight) will shrink or grow based on which stocks have moved enough to no longer be "under $20."

4. **The deep-dive pool stays top-30.** 427 of 457 candidates were never individually scored tonight. That ratio will be similar Monday.

5. **CYCN, FRMM, BIRD are already in the persistent DB** and will be offered for execution without needing to re-enter the deep-dive pool — unless their DB status changes (e.g., if they're marked "traded" or "dead").

---

## Anomalies & Flags

**⚠ SCORING ADJUSTMENT MISMATCH:** The learning loop writes `data/v2_scoring_adjustments.json` with keys `composite_score`, `float_shares`, `short_pct`, `has_catalyst`. The scoring engine reads `data.get("multipliers", {})`. No `multipliers` key exists → all adjustments are 1.0. The 8.3% win rate the learner saw 6 hours ago prompted adjustments that are being silently ignored. Every scan since the learning loop was activated has run at default weights.

**⚠ FRMM bypasses anti-chase at +81.8% gap.** The bypass is intentional (times_seen=4), but this stock has no identified catalyst. It's a trifecta squeeze play that's already moved 139% in 5 days. Entering Monday morning at any price level that maintains or exceeds Friday's close means chasing a stock with a 139% 5-day gain and no known news driver.

**ℹ BIRD seen 22 times.** It has been in the research DB watching list for an extended period. Vol_accel of 376x is extreme — that's 3-day average vs 10-day, meaning Friday's volume was extraordinary vs the prior 10 days. This is likely the Finviz scan1 + Alpaca gainer crossover that kept it returning each session.

**ℹ Alpaca gainers include many warrants.** ORIQW, BZAIW, SVREW, CRMLW, SEATW, DAVEW etc. These pass `len(sym) <= 5` when the warrant suffix is W as a 5th character. They never make it into the research DB because the engine doesn't explicitly filter warrant suffixes beyond the dot-check (`"." not in sym`).
