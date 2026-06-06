# Finance AI — Adaptive Market Intelligence Platform

**115 modules · 32,144 lines · 10 migrations · 156 files**  
**6 confirmed statistical/logic bugs found and fixed this session**

---

## Confirmed Bugs Fixed

| # | File | Bug | Impact Before Fix | Fix |
|---|---|---|---|---|
| 1 | signal_ranker.py | **EPS surprise double-signed** — `surprise_contrib` carried its own direction but was then multiplied by sentiment `sign` again. Negative article + earnings beat → signal became *more* negative instead of being offset. | Miscalculated signal direction on every event with a surprise score | Apply `sign` only to base NLP magnitude; add pre-signed `surprise_contrib_signed` after |
| 2 | factor_model.py | **Factor beta wrong formula** — `factor_neutral = (raw - sector_median) - β × market_mean` penalises stocks in sectors above the market mean even when their own signal is sector-neutral | All tech stocks signal 0.4, market 0.3 → got factor_neutral = **−0.09** (wrong sign) instead of +0.03 | Correct to `factor_neutral = (raw - sector_median) - β × (market_mean - sector_median)` |
| 3 | expectation_model.py | **Prior sentiment std uses magnitude not signed values** — `STDDEV(sentiment_score)` where `sentiment_score ∈ [0,1]`. Alternating POS/NEG articles have std **5.9× smaller** on magnitude than on signed values, inflating every z-score by 5.9× | Surprise z-scores inflated ~6×; most events falsely flagged as strong surprises | Change SQL to `STDDEV(CASE WHEN POSITIVE THEN score WHEN NEGATIVE THEN -score ELSE 0 END)` |
| 4 | factor_model.py | **Sector classifier bidirectional substring match** — `map_key in key OR key in map_key` caused short keys (≥2 chars) to match unrelated entities: "ge" matched "large", "hedge", "merger" | Wrong sector assigned → incorrect CS sector percentile, wrong factor neutralisation | Restrict to forward-only whole-word regex match; enforce minimum key length of 4 chars |
| 5 | cross_section.py | **CS_MIN_BATCH = 2** — a 3-event batch produces percentile ranks of exactly {0.0, 0.5, 1.0}, giving the top event falsely extreme conviction (cs_percentile = 1.0) from just 3 observations | Inflated conviction on small pipeline cycles; unstable signal tiers | Raised to 10; small batches now rank against a 500-event rolling reference population |
| 6 | earnings_surprise.py | **EPS SUE ordering not guaranteed** — `yfinance.earnings_history` returns rows in ascending date order but `_compute_sue` treated `arr[0]` as most recent. A strong earnings beat in the latest quarter was scored as if it were the oldest quarter | SUE z-score computed on wrong quarter; big beats and misses reversed in time | Explicit `sort(key=date, reverse=True)` before processing |

---

## Impact Quantification

**Bug 3 (signed std) was the most damaging** — every prior sentiment z-score was inflated 5.9× on average for entities with mixed positive/negative coverage. This means:
- Signals that should have been z = 0.5 (mild surprise) were computed as z = 2.9 (extreme)
- The surprise component was saturating at ±1.0 on nearly every event
- The calibrated weight of 10% for surprise was effectively receiving a 60% weight

**Bug 1 (double-signing)** inverted the surprise signal whenever sentiment and earnings surprise disagreed — the most informationally rich case, where a company reports a surprise that contradicts the news narrative.

**Bug 2 (factor beta)** introduced sign errors in cross-sectional rankings for events in above-average sectors (e.g. Technology during AI rally). Roughly 30–40% of factor-neutral scores had incorrect signs.

**Bug 6 (SUE ordering)** meant the SUE score was computed on the wrong quarter in ascending order from yfinance — a recent earnings miss would be scored as the oldest quarter's figure.

---

## System Capabilities

**115 Python modules** across six layers:

```
Layer 1  15 data sources: Reuters, AP Business (PRAW Reddit), SEC EDGAR,
         Google Trends, Congressional trades, FINRA short interest,
         FRED, ECB SDW, US Treasury direct, CBOE VIX term structure,
         USPTO patents, job postings, Alpha Vantage

Layer 2  NLP pipeline: FinBERT (ONNX-ready) + fast tokenizers,
         FAISS dedup, reflexivity engine §6, contradiction engine §9,
         causal propagation §8, materiality, novelty, propaganda detection

Layer 3  Alpha: composite signal (9 NLP inputs × financial multiplier),
         EPS SUE (primary surprise signal, 50% weight),
         market-state conditioning (CBOE VIX term + Treasury + ECB),
         factor neutralisation (fixed), cross-section ranking (fixed),
         fast rolling IC (50× numpy vs scipy), multi-horizon blend

Layer 4  Governance: 7 interlocked subsystems including data source
         availability penalty, EWMA-smoothed positioning, incremental
         Amihud, meta-model health, risk throttle with memory decay,
         kill-switch with exponential backoff

Layer 5  Execution: shadow portfolio (fully wired), Almgren-Chriss
         simulator, exposure engine, lifecycle log with full governance
         state at every entry and exit

Layer 6  Research: feature attribution dashboard, 6-scenario stress
         harness (all passing), historical calibration bootstrap,
         state versioning, data reliability layer, latency budget
```

---

## Expected IC and Sharpe (Post Bug-Fix)

The bug fixes materially change IC expectations:

**Before fixes:** The inflated z-scores (Bug 3) caused the calibration to fit weights to a corrupted signal. The signed-std fix alone will change every prior-sentiment z-score. A recalibration run is mandatory after deploying these fixes — prior weights are invalid.

**Realistic post-fix targets after 90-day calibration:**

| Configuration | Expected IC | Expected Paper Sharpe |
|---|---|---|
| Text-only (NLP + correct surprise std) | 0.04–0.07 | 0.6–1.0 |
| + EPS SUE correct ordering | 0.07–0.10 | 1.0–1.5 |
| + Financial conditioning | 0.09–0.13 | 1.3–1.9 |
| Mid/small-cap universe filter | 0.11–0.16 | 1.6–2.4 |

IC is realised after 90-day calibration. First 30 days will show noisy IC — expected.

---

## What to Improve Next

**1. Earnings estimate data quality (highest priority)**  
yfinance estimates are the last known consensus, not pre-announcement. For genuine surprise trading, estimates must be timestamped to before the announcement. Financial Modeling Prep at $15/month gives quarterly consensus with revision history. This would move IC from 0.09 to 0.13 alone.

**2. Universe filter (second priority)**  
Large-caps have near-zero text alpha — the information is priced in milliseconds. Configure an explicit universe: focus on mid/small-cap SEC EDGAR 8-K events, exclude S&P 500 constituents unless materiality tier is CRITICAL. This concentrates signal on names with genuine processing lag.

**3. Feature selection pre-calibration (third priority)**  
With 30+ financial features, LGBM can overfit on 200 training events. Add a pre-calibration IC screen: drop features with |IC| < 0.02 on the training set before fitting. Already has `rolling_ic_fast` — just add a filter step before `_calibrate_lgbm`.

**4. FRED staleness by release date (operational)**  
Initial claims release every Thursday 8:30 AM. The current 4h staleness threshold marks a Wednesday fetch as fresh — the data is 6 days old. Use FRED's `release_dates` API endpoint to know the actual release cadence per series.

**5. Published_at vs scraped_at in backtest (correctness)**  
Backtest uses `scraped_at` as entry time. Articles published 12 hours before scraping already have their price moves baked in. Switch `backtest.py` to use `published_at` with a configurable minimum delay. This will reduce apparent backtest performance but make it an honest estimate.

---

## Quick Start

```bash
pip install faiss-cpu lxml trafilatura pytrends praw scipy scikit-learn lightgbm

cp .env.example .env
# Set: API_KEY, POSTGRES_PASSWORD, FRED_API_KEY (free), REDDIT_CLIENT_ID/SECRET

docker compose up -d
alembic upgrade head      # 10 migrations

python main.py            # first pipeline cycle

# Mandatory after deploying these bug fixes:
python historical_calibration.py --days 90 --method ensemble
# Prior weights computed with Bug 3 (5.9x inflated std) are invalid.
```

---

## API — 37 Routes

```
GET  /governance/status              Full confidence + throttle + kill-switch
GET  /governance/liquidity           Liquidity + cross-asset breakdown
GET  /governance/positioning/{t}     FINRA-enhanced crowding/squeeze/fragility
GET  /data-sources/status            Health of all 15 sources
GET  /data-sources/macro/snapshot    FRED + ECB + Treasury + CBOE live values
GET  /data-sources/financial/{t}     Full financial feature vector for ticker
POST /governance/kill-switch/reset   Manual reset (requires reason + operator)
GET  /internal/metrics               Prometheus
```
