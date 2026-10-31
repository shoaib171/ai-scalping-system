# Handoff — AI Trading System (from Mac Claude Code session, 2026-10-28)

Read this first. It carries everything decided/learned in the Mac session so the Windows session can continue.

## User profile
- Trades **gold (XAUUSD)** manually with **SMC**: liquidity sweep → 1m/5m CHoCH/MSS → FVG/OB entry. Target **1:3 RR**. Typical size **0.1 lot (10 oz), ~$50 risk**.
- Sessions: **London open → 22:00 PKT** (incl. London/NY overlap). Times always in **PKT (UTC+5)**.
- Communicates in **Roman Urdu**; reply in Roman Urdu.
- Has built many EAs before; considers them a waste of time because they were strategy-first. **Now wants data-first**: data → deep-learning prediction model → then strategy/EA.
- Windows laptop has **MT5 (account logged in) + Claude Code**. Mac has TradingView Desktop (OANDA feed) connected via tradingview-mcp (CDP port 9222).

## Project goal (stated by user)
An MT5 trading system built with **Python / Node / React** (system design to be decided):
1. **Data first**: MT5 price history (XAUUSD, XAGUSD, EURUSD, DXY), **ForexFactory calendar as JSON**, **news headlines** via user's GitHub repo called **"open terminal"** (link/path NOT yet provided — ask for it first).
2. **Deep-learning prediction model** on that data.
3. **SMC-based daily scalping + intraday** trades.
4. **Auto mode by default**; **manual mode** where the user **chats with the bot** (like this Claude chat) and asks it to place entry/SL/TP. Rules for each mode to be defined.
5. Process the user demanded: **first complete tech + non-tech requirements and system logic**, list all features, then per feature: requirements → MT5 connection → use cases → test cases → tested → next feature. **No code until requirements/spec are approved.**

## Proposed decomposition (presented, user has not objected)
| # | Sub-project | Depends on |
|---|---|---|
| 1 | Data layer (MT5 history + live, FF calendar JSON, headlines, storage) | — |
| 2 | Research & ML (features, labels, DL model, walk-forward validation) | 1 |
| 3 | Signal engine (SMC detectors + model filter + rules, session London→22:00 PKT) | 1,2 |
| 4 | MT5 bridge / execution | — |
| 5 | Risk & mode manager (auto default / manual, limits, news filter, kill switch) | 3,4 |
| 6 | Chat interface (LLM, manual-mode order entry) | 4,5 |
| 7 | Dashboard (React) | all |
Start with 1 + 2, but capture high-level requirements of all parts first.

**Constraint:** MetaTrader5 Python package is Windows-only → develop on the Windows laptop (or Windows VPS). Alternative: MQL5 EA ↔ backend over sockets.

**Open question to resume with:** link or local path of the "open terminal" GitHub repo → read it fully to learn what data it gives (headlines? calendar? format? latency?). Then continue requirements one question at a time.

## Research findings so far (TradingView OANDA data)
**Correlation test, 15 days 5m (~3.3k bars) + ~9 days 1m (~10.6k bars):**
- Same-bar return correlation: EURUSD–DXY **−0.95** (5m)/−0.84 (1m); DXY–XAU −0.46/−0.25; EURUSD–XAU +0.44/+0.32; **XAG–XAU +0.82/+0.80**.
- Chain (EUR strong up & DXY strong down same bar) → gold same bar expected dir **75% (5m) / 69% (1m)**, but **next bar ~50%** → relationships are **simultaneous, not sequential**. Lead-lag correlations all within noise (|r| < 0.07).
- "Silver leads gold": silver-first 20-bar breakout → gold follows 49–50% vs gold-first → silver 46%; trading gold after silver-first break = ~$0 edge (5m: −$0.91 avg over 10 bars) → **not tradable after $0.30 spread** at 1m/5m resolution. Might exist at tick level (untested).
- 5m gold 20-bar breakouts mean-reverted in this period regardless of silver confirmation (SMT −$1.00 vs both-break −$1.46 over 10 bars).
- **Implication:** use DXY/EUR/XAG as a **confirmation filter on SMC setups** (to be tested on 6–12 months of MT5 data), not as a standalone predictor. Prefer ML **meta-filter** ("will this SMC setup win?", start with gradient boosting) over raw next-candle price prediction (overfits).

**Other backtests:**
- User's Pine "S/R Rejection + M1 Structure Breakout" (19–27 Oct, 1m): 21 trades, 23.8% win, −4.08%, PF 0.62; commission 0.02%/side ate ~$357. Needed leverage set (Pine v6 default 1x → 0 trades). Bias logic flaw: supportHolding doesn't require touching the zone.
- Manual SMC review 27 Oct: 3 A-setups each +3R (London Asian-low sweep reversal; internal EQL sweep scalp; NY PDH break → 5m FVG retest). Rules correctly skipped a PDH-sweep short with no CHoCH.

## SMC detection definitions agreed in chat (basis for signal engine)
- Swing H/L: fractal, N=2–3 bars each side; usable only after confirmation (no repaint).
- Liquidity: PDH/PDL, Asian H/L (00–07 UTC), London H/L (07–12 UTC), EQH/EQL (tolerance ≈ $1.5 15m / $2 1H / $4 4H, unbroken), internal 15m swings.
- Sweep: wick ≥ $0.20 beyond level + body close back inside within ~5–10 bars; body acceptance beyond = breakout, not sweep.
- BOS/CHoCH: **body close** beyond confirmed swing. Displacement: body ≥ 1.5×ATR leaving an FVG. FVG: `low[i] > high[i-2]` (bull). OB: last opposite candle before the break; fresh = untouched.
- Premium/discount around 50% of dealing range; OTE 62–79%.
- Spread matters: sell SL triggers on **ask** while chart shows bid → place SL beyond zone high + spread + buffer.

## Lessons / pitfalls
- tradingview-mcp: `pine new` can fail silently and `pine compile` may click **Save**, overwriting the open saved script (happened to "SMC Signal Suite"; restored via Version history). Always check what's open in Pine Editor first.
- User asked to remove all their strategies/rules from the old project; their saved TV scripts (Algo bot Scalping, DXY Correlation Scalper Signal, SMC Signal Suite) still exist — deletion is theirs to do.
- TradingView only loads ~1–9 days of 1m history → use MT5/Dukascopy for long history.
