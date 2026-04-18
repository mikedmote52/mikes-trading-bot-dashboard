# Scanner Config Snapshot
**Generated:** 2026-04-17  
**Interpreter:** `/usr/local/bin/python3` → Python 3.13.5 (Python.framework/Versions/3.13)  
**Daemon plist:** `~/Library/LaunchAgents/com.mikes-trading-bot.plist`  

---

## 1. Finviz Screener — Four Scans

All four scans use `finvizfinance.screener.overview.Overview`. Source: `v2_research_engine.py:273–403`.

### Scan 1 — Short Squeeze (v2_research_engine.py:289–294)
```python
filters = {
    "Market Cap.":  "+Micro (over $50mln)",
    "Float Short":  "Over 10%",
    "Relative Volume": "Over 2",
    "Price":        "Under $20",
}
```
Meaning: Micro-cap ($50M+), short float >10%, rel volume >2x, price <$20.

### Scan 2 — Unusual Volume (v2_research_engine.py:315–320)
```python
filters2 = {
    "Market Cap.":  "+Small (over $300mln)",
    "Relative Volume": "Over 5",
    "Price":        "Under $20",
    "Change":       "Up",
}
```
Meaning: Small-cap ($300M+), rel volume >5x, price <$20, up on the day.

### Scan 3 — Biotech (v2_research_engine.py:345–350)
```python
filters3 = {
    "Industry":     "Biotechnology",
    "Market Cap.":  "+Micro (over $50mln)",
    "Relative Volume": "Over 2",
    "Change":       "Up",
}
```
Meaning: Biotech only, micro-cap, rel volume >2x, up on the day.

### Scan 4 — Pre-Catalyst (v2_research_engine.py:373–378)
```python
filters4 = {
    "Float Short":  "Over 15%",
    "Market Cap.":  "+Micro (over $50mln)",
    "Price":        "Under $20",
}
```
Meaning: Short float >15%, micro-cap, price <$20. **No volume requirement** — catches setups before they move.

**Dedup logic:** Each scan adds only tickers not already in the candidates set. Priority order for deep-dive is Finviz source first, then social mentions.

---

## 2. Alpaca Movers — Two Calls

Source: `v2_research_engine.py:514–549`

```python
# Gainers
GET https://data.alpaca.markets/v1beta1/screener/stocks/movers?top=50
# Returns: data["gainers"][].{symbol, percent_change, volume, trade_count}

# Most-Actives
GET https://data.alpaca.markets/v1beta1/screener/stocks/most-actives?by=volume&top=50
# Returns: data["most_actives"][].{symbol, volume, trade_count}
```

Filter applied to both: `"." not in symbol` and `len(symbol) <= 5` (excludes warrants, ADRs with dots).  
Max returned: 50 gainers + up to 50 most-actives (deduplicated).

---

## 3. Reddit Scan

Source: `v2_research_engine.py:555–639`

**Subreddits (5):** `pennystocks`, `squeezeplays`, `shortsqueeze`, `wallstreetbets`, `stocks`  
**Posts per subreddit:** 50 latest (`?limit=50` via `old.reddit.com`)  
**Ticker patterns:**
- `$SYMBOL` (2–5 chars, dollar-prefixed) — trusted regardless of length
- bare `WORD` (4–5 chars only) — bare words, less ambiguous

**Ignore list:** 100+ common non-ticker strings (THE, AND, CEO, IPO, SPY, VIX, FOMC, FDA, etc.) `v2_research_engine.py:569–600`

**Mention threshold to enter all_candidates:** ≥ 3 (line 1035)  
**Mention threshold to appear in trending report:** ≥ 2 (line 637)

---

## 4. Deep-Dive Selection

Source: `v2_research_engine.py:1042–1044`

```python
priority_order = sorted(all_candidates.values(),
    key=lambda x: (x.get("source","").startswith("finviz"), x.get("social_mentions",0)),
    reverse=True)[:30]  # Deep dive on top 30
```

Top 30 candidates are selected, prioritizing Finviz-sourced candidates, then by social mention count. Finviz detail calls are limited to the first 20 (`i < 20` check, line 1052).

---

## 5. Scoring Function — Factor Weights

Source: `v2_research_engine.py:645–917`

### Catalyst (lines 664–698)
| Signal | Score |
|--------|-------|
| Bullish news (<48h) | +20 × adj["catalyst"] |
| Bearish news (<48h) | -20 × adj["catalyst"] |
| Neutral/mixed news (<48h) | +10 × adj["catalyst"] |
| Biotech/pharma sector | +15 × adj["catalyst"] |

### Float (lines 700–720)
| Float | Score |
|-------|-------|
| < 10M shares | +30 × adj["low_float"] |
| 10M–30M | +20 × adj["low_float"] |
| 30M–50M | +10 × adj["low_float"] |
| > 50M | 0 |

### Short Interest (lines 722–746)
| SI % of float | Score |
|---------------|-------|
| > 20% | +30 × adj["high_short"] |
| 10%–20% | +20 × adj["high_short"] |
| 5%–10% | +10 × adj["high_short"] |

### Volume Acceleration (yfinance 3d vs 10d avg) (lines 748–773)
| Volume accel | Score |
|--------------|-------|
| > 5x | +25 × adj["volume"] |
| 3x–5x | +15 × adj["volume"] |
| 2x–3x | +10 × adj["volume"] |

### Alpaca intraday % change (lines 762–773)
| Day pct change | Score |
|----------------|-------|
| > 50% | +20 × adj["volume"] |
| 20%–50% | +15 × adj["volume"] |
| 8%–20% | +10 × adj["volume"] |

### Social Mentions (lines 781–810)
| Mentions | Sentiment | Score |
|----------|-----------|-------|
| ≥ 10 | positive (>0.3) | +20 × adj["social"] |
| ≥ 10 | negative (<-0.3) | -10 × adj["social"] |
| ≥ 10 | neutral | +10 × adj["social"] |
| 5–9 | positive | +10 × adj["social"] |
| 5–9 | negative | -5 × adj["social"] |
| 5–9 | neutral | +5 × adj["social"] |
| 2–4 | positive | +5 × adj["social"] |
| 2–4 | negative | -3 × adj["social"] |
| 2–4 | neutral | +2 × adj["social"] |

### Price Sweet Spot (lines 824–837)
| Price | Score |
|-------|-------|
| $2–$10 | +10 × adj["price_sweet_spot"] |
| < $2 | -10 × adj["price_sweet_spot"] |
| > $20 | -5 × adj["price_sweet_spot"] |

### No-Catalyst Penalty (lines 840–856)
```python
# Applied when catalyst_score <= 0:
score -= 30 * adj["no_catalyst_penalty"]

# Compound pump-and-dump penalty (no catalyst AND no squeeze AND vol_accel > 2):
score -= 15 * adj["no_catalyst_penalty"]
```

### Score Normalization (lines 862–867)
```python
max_possible = 150
score = min(100, max(0, int((raw_score / max_possible) * 100)))
```

### Squeeze Trifecta Bonus (lines 872–881) — applied AFTER normalization
```python
if _float_m < 15 and _si_pct > 15 and vol_accel > 5:
    trifecta = True
    score = min(100, score + 40)
```
Float < 15M shares AND SI > 15% AND volume acceleration > 5x → +40 to normalized score, forces S tier.

---

## 6. Conviction Tiers

Source: `v2_research_engine.py:883–892`

| Tier | Condition |
|------|-----------|
| S | Trifecta triggered OR normalized score ≥ 75 |
| A | score ≥ 55 |
| B | score ≥ 35 |
| C | score ≥ 20 |
| PASS | score < 20 |

**Trade eligibility (run_bot.py:832–837):** Must be tier S/A/B AND composite_score ≥ 40.

**Score-to-tier mapping for execution (run_bot.py:366–374):**
| v2 score | Execution tier |
|----------|----------------|
| ≥ 80 | S |
| ≥ 60 | A |
| 40–59 | B |
| < 40 | C (no trade) |

**DB read threshold:** `get_trade_candidates()` requires `ai_score >= 50` (v2_research_engine.py:982).

---

## 7. Anti-Chase Filter

Source: `run_bot.py:914–933`

```python
if prev_close_snap > 0:
    gap_from_prev = ((limit_price - prev_close_snap) / prev_close_snap) * 100
    if gap_from_prev > 15:              # 15% gap threshold
        sym_times_seen = times_seen_map.get(pick["symbol"], pick.get("times_seen", 1))
        on_premarket_wl = pick["symbol"] in premarket_wl_symbols
        if sym_times_seen <= 1 and not on_premarket_wl:
            # SKIP — anti-chase
```

**Gate triggers:** gap from prev close > 15%.  
**Exemptions (bypass the gate):**
- `times_seen > 1` — stock seen on ≥ 2 prior scans (in research DB)
- `on_premarket_wl = True` — in the premarket watchlist file

---

## 8. Position Sizing

Source: `run_bot.py:877–884`

| Tier | Dollar allocation | Stop level |
|------|------------------|------------|
| S | $100 | -20% |
| A | $75 | -15% |
| B | $50 | -10% |

**Limit order pricing (run_bot.py:906–908):**  
`limit_price = round(ask * 1.01, 2)` — 1% above live ask.

**Daily constraints (run_bot.py:68–71):**
- `DAILY_BUDGET = $300`
- `MAX_PER_TICKER = $100`
- `MAX_POSITIONS = 20`

---

## 9. Exit Rules

Source: `run_bot.py:73–78` and `trading/position_monitor.py:79–90`

| Rule | Threshold | Priority |
|------|-----------|----------|
| Hard stop | -10% from entry (B tier) / -15% (A) / -20% (S) | Highest |
| Trailing stop | -10% from peak | Activates after +20% gain |
| Profit target | Sell half at +50% | — |
| Time exit | 30 trading days | — |
| Squeeze broken | RVOL < 1.0 AND SI < 5% | Lowest |

---

## 10. Circuit Breakers

Source: `run_bot.py:80–82`

```python
MAX_CONSECUTIVE_LOSSES = 3
MAX_DAILY_DRAWDOWN_PCT = 10.0
```

Circuit breaker activates on 3 consecutive losses OR 10% daily drawdown. Resets next trading day.

No stop-out cooldown window beyond the per-session circuit breaker. No multi-session cooldown implemented.

---

## 11. Scoring Adjustments (Learning Loop Output)

Source: `data/v2_scoring_adjustments.json` — read by `v2_research_engine.py:57–78`

**⚠ MISMATCH BUG:** The learning loop writes adjustments with keys like `composite_score`, `float_shares`, `short_pct`. The scoring engine reads `data.get("multipliers", {})` and expects keys like `catalyst`, `low_float`, `high_short`. Since no `multipliers` key exists in the file, **all scoring adjustments default to 1.0 — the learned weights are silently not applied.**

What the learning loop last computed (from `data/v2_scoring_adjustments.json`, run 2026-04-17):
```json
{
  "composite_score": { "multiplier": 1.2 },   // +20%
  "float_shares":    { "multiplier": 0.8 },   // -20%
  "short_pct":       { "multiplier": 0.9875 },// -1.25%
  "has_catalyst":    { "multiplier": 0.95 }   // -5%
}
```
Win rate at time of last learning run: 8.3% (12 trades analyzed).

What the scoring engine actually uses: **all multipliers = 1.0** (defaults).

---

## 12. Sector Bonus (scoring_engine.py)

Source: `scanner/scoring_engine.py:460–466`

Hot sectors: `["AI", "quantum", "biotech", "energy", "defense"]`  
Bonus: +15 to composite score.

Note: This is the `ScoringEngine` used by `scanner/pipeline.py` (the Polygon-based 8-stage pipeline). The **production daemon calls `v2_research_engine.run_research()`**, NOT `scanner/pipeline.py`. The pipeline file is not wired into the live execution path.
