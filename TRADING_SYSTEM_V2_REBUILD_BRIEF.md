
# Trading System v2 Rebuild Brief
**Date authored:** 2026-04-17
**Author context:** Written after a deep audit of mikes-trading-bot + openclaw-trader
**Intended reader:** A fresh Cowork/Code session picking this work up cold

---

## Read this first

You are picking up an in-flight rebuild of Mike's autonomous trading system. The current production system is live at `/Users/michaelmote/Desktop/mikes-trading-bot` with a dashboard publishing to https://mikedmote52.github.io/mikes-trading-bot-dashboard/. An adjacent scanner codebase lives at `/Users/michaelmote/Desktop/openclaw-trader`. Both are currently running under launchd. Do not break them.

Before you write any code, read this entire brief and then read these three files in order:
1. `closed_trades.json` in this repo (or wherever the bot writes it)
2. `run_bot.py` focusing on `reconcile_pending_fills()` at line ~589
3. `diamond_scanner.py` in openclaw-trader, focusing on the `DYNAMIC_WEIGHTS` loader

The brief tells you what to build and why, in strict dependency order. Do not skip ahead. Shifts 3 through 6 produce zero value if Shifts 1 and 2 are incomplete.

When you finish a shift, commit, push, run the smoke test, and only then move to the next. If a smoke test fails, stop and fix — don't patch over it in the next shift.

---

## The diagnosis

The current system runs two independent scanners that don't feed each other. OpenClaw's `diamond_scanner.py` optimizes a weighted composite for stocks that will appear on end-of-day movers lists. The bot's `v2_research_engine.py` scores candidates 0–100 through a parallel research pipeline. The bot places tiered positions with server-side GTC hard stops that swap to trailing stops at +20% gain. On paper this is a reasonable small-cap momentum system with sensible risk controls.

In practice the system has made 228 trades and learned nothing. Win rate 35.5%, average return -1.95%. Hard stops have hit 28 times at an average loss of 18.77%. Trailing stops have hit 7 times at an average gain of 20.87%. Those numbers don't sum to a profitable strategy. They sum to a strategy bleeding out in its tail.

The reason the system isn't correcting its own behavior is a data pipeline break disguised as working infrastructure. Entries record rich factor flags (has_catalyst, float_shares, short_pct, volume_acceleration). `reconcile_pending_fills` in `run_bot.py` joins Alpaca fills back to entries at exit, but does not carry those factor flags into `closed_trades.json`. The nightly learner (`nightly_learner.py`) exists, is well-written, and is never called from the bot daemon. `v2_adjustments.json` is loaded by the research engine on every scan, but nothing has ever written to it. OpenClaw's `scanner_weights.json` likewise does not exist on disk; the scanner's dynamic-weights loader falls through to inline defaults every run. The `learning_eval_report.json` shows `fired = 0` across every signal for every one of the 228 trades. The correlation math works. The data never arrives.

This is the root failure. Everything else cosmetic.

A separate and also important problem: OpenClaw and the bot are not actually integrated beyond display. The bot writes portfolio state to `openclaw-trader/state/current.md` every five minutes so OpenClaw's web UI can render it. OpenClaw writes nothing back. `diamonds.json` is never consumed by the bot. The bot runs its own research. If you replaced OpenClaw with /dev/null today the bot would not notice.

---

## North star

Three outcomes, ranked.

**Outcome one: closed feedback loop.** Every decision recorded with enough context that the nightly learner can correlate it against outcome and update weights. When a weight changes, the change and its reason land in a journal.

**Outcome two: tail-shape improvement.** Compressing hard-stop losses, expanding trailing-stop winners. Aim: in three months, hard-stop losses average -10% instead of -18.77%, and trailing-stop exits represent 10%+ of closed trades instead of 3%.

**Outcome three: cognition-first dashboard.** Answer three questions at a glance: are we winning or losing this month, why, and what should change. Surface tail shape, cohort performance, decision funnel, portfolio attribution, weight drift, regime. Not a list of tickers.

If a change doesn't move one of these, defer it.

---

## Sequenced plan

Execute in order. Each shift has a definition of done. Do not start the next shift until the previous is committed, pushed, and passing its smoke test.

### Shift 1 — Close the schema loop

Modify `reconcile_pending_fills` in `run_bot.py` (~line 589) to read from `research_v2.db` using symbol and entry timestamp as join key. Pull every scoring input: tier, composite_score, has_catalyst, catalyst_type, float_shares, short_pct, volume_acceleration, sector, gap_pct, rvol, previous_close. Write them into `closed_trades.json` as an `entry_signals` sub-object along with a thesis snippet.

If `research_v2.db` is missing fields, write a per-entry `decisions/decision_{timestamp}_{symbol}.json` at buy submission. Reconciler reads that on exit.

**Done when:** A paper trade fills, closes, and the resulting `closed_trades.json` row has every field populated non-null with a thesis snippet. No schema warnings.

### Shift 2 — Wire the nightly learner

Add a 5:30 PM ET daily hook in the bot daemon that calls `NightlyLearner.run_nightly(closed_trades_path, output_path='v2_adjustments.json')`. Append every weight change >1% to `journal/weight_drift.jsonl` with timestamp, signal, old/new weight, sample size, rationale. Verify the research engine reads `v2_adjustments.json` and propagates multipliers.

Do the same loop on OpenClaw. Fix the existing path bug (`/Documents/openclaw-trader/` should be `/Desktop/openclaw-trader/`). OpenClaw learns from bot outcomes, not market gainers.

**Done when:** Three trading days post-Shift-1, `v2_adjustments.json` and `scanner_weights.json` have fresh mtimes nightly, `weight_drift.jsonl` has real entries, and a scan reflects adjusted multipliers.

### Shift 3 — Replace the proxy objective

Change OpenClaw's target surface from "will this appear in end-of-day movers" to "would this have produced a positive bot outcome." Build a labeling function that computes realistic entry and exit paths for historical candidates and labels them profitable/unprofitable. Train weights against that label.

Add execution-aware features: intraday spread quality at signal, realized volatility in the hour before signal, correlation to held positions. Retire or zero-weight raw float (covered in volume dynamics), pure market cap, and LLM-budget-exhausted catalyst keyword fallback.

**Done when:** 4-week backtest or live-forward shows bot-traded scanner picks produce positive expected return independent of stop changes.

### Shift 4 — Re-tune the stop regime from data

Analyze every hard-stop exit for realized max drawdown before stop and post-stop 24-hour path. If meaningful recovery after stop-out, widen tier-specific stops (consider S=-25%, A=-20%, B=-15%). Analyze the 193 "other" exits for time-based chop in the tail. Consider raising minimum composite score for entry if B-tier shows negative expected return.

**Done when:** `stop_tuning_analysis.md` lands with proposed changes and expected-P&L impact from historical replay. Config change lands only after review.

### Shift 5 — Correlation-aware portfolio construction

Before entry, compute 20-day return correlation between candidate and each held position. Reject entries with any correlation > 0.7. Add portfolio effective-count check: sum of 1/(1+correlation_to_portfolio_mean). If effective count < 8, reject new correlated entries.

**Done when:** Correlation check runs and logs on every buy; 4-week simulation shows rejection count and realized volatility impact.

### Shift 6 — Cognition-first dashboard

Redesign around two tabs.

**Month tab:** MTD P&L curve with target line and days-remaining as hero. Cohort breakdown table (tier, score range, catalyst, float class, short-interest class) with count / win rate / avg return. Portfolio attribution with correlation highlighted.

**Mind tab:** Decision funnel for today (candidates in → anti-chase killed → score killed → trifecta cleared → bought) with conviction distribution on survivors. Weight drift journal. Regime gauge top-right: small-cap momentum, realized vol percentile, market breadth. Red/yellow/green.

Drop ticker-centric tabs. Positions are click-throughs, not primary view.

Add a `regime.json` writer: Russell 2000 20-day return, SPX 20-day realized stdev, NYSE advance-decline. Daily pre-open.

**Done when:** Two tabs. Empty panels say "insufficient data" rather than faking. User at 9 AM ET knows in 30 seconds whether to trade and which signals produce edge.

---

## Hard rules

1. Don't skip Shift 1. Everything correlates nulls without it.
2. Don't ship UI before data exists. Gate every panel.
3. One shift per commit. Smoke test passes before next shift.
4. No feature work during rebuild. Log discoveries to `REBUILD_FOLLOWUPS.md`.
5. Paper trade through Shift 2. Live with reduced size from Shift 3.
6. Launchd only. No cron.

---

## When you're done

Month over month:
- MTD P&L curve trends toward target
- Cohort table shows clear positive and negative edge buckets
- Weight drift journal shows modest explainable weekly adjustments
- Regime gauge correctly flags sit-out days
- Hard-stop tail compressed below -18.77%
- Trailing-stop exits above 3% of closed trades

If two or more aren't true by month-end, the loop is broken again — diagnose before adding features.

---

## Open questions for Mike

1. Paper only through Shift 2, then live at reduced size? Recommended.
2. Fold OpenClaw into bot's v2_research_engine, or keep separate? Recommend fold.
3. Monthly P&L target for dashboard target line?
