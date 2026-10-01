# SPEC — Micro Commodity Trend Bot (PAPER ONLY)

Status: **Gate 1 (spec) APPROVED — 2026-10-01.** Kelly mapping deferred (Kelly disabled).
Amended 2026-10-01 during Gate 2 (user decisions): §5 daily-close definition, fill timing and slippage sensitivity gate; §6 Jev shadow mode and pre-registered activation criteria.
Handoff target: Claude Code. Next: Gate 2 (architecture), which needs user approval before any plan or code.

---

## 0. Scope decisions

- ✅ **Paper trading only.** Real account size (£100) cannot support any futures position under the risk rules. Live trading is out of scope for this build.
- ✅ **History:** 20+ years of daily data, walk-forward evaluation.
- ✅ **Simulated equity:** IBKR paper balance £200,000. All risk, margin and sizing maths use this figure. It is configured, never inferred.
- ✅ **Base currency: GBP.** Contracts are USD-denominated: convert P&L, risk and margin at the IBKR FX rate and log the rate used.

### Paper-only enforcement (hard-coded, not configurable at runtime)
- Connect only to the IB Gateway paper port. Refuse to start on any other port.
- On connect, read the account ID and **halt if it is not a paper account** (IBKR paper IDs start with `DU`; verify this against the connected account rather than trusting it).
- No code path places orders to a live account. Going live later requires a separate spec, separate approval, and real funding that meets section 7.

---

## 1. Architecture (three non-overlapping layers)

| Layer | Role | Cadence | May never |
|---|---|---|---|
| Opus 5.5 | Strategy design, backtests, code, nightly review | Nightly / on demand | Grade its own output; touch limits, sizes, orders |
| Jev (TypeSafe, `typesafe-sdk`) | Score fixed-outcome questions on a compact state | Once per daily bar per market | Create a trade; override any limit |
| Deterministic code | State, rolls, thresholds, sizing, risk vetoes, orders | Always | Delegate any decision above to a model |

Build harness: verify the AgenKit install source before use (could not confirm agenkit.xyz). Approval pauses at spec, architecture, plan. Every module: failing test first, review against this spec, rollback.

Secrets: Jev key in `.env` only (user enters it). Never request, store or automate IBKR password/2FA or Gateway login.

---

## 2. Markets

MGC (Micro Gold), SIL (Micro Silver), MCL (Micro WTI), MHG (Micro Copper). Micros only.
Multiplier, tick size/value, trading hours, margin, settlement type, first notice day and last trade date are **read from IBKR at runtime**, never hardcoded. Gold and silver are **one correlated bucket** for all limits.

---

## 3. Data

- ✅ 20+ years daily, covering 2008, 2014–15, 2020 (incl. negative WTI), 2022.
- Backtests use **full-size proxies** (GC, SI, CL, HG) scaled to micro multipliers, with micro commissions and tick slippage. Micro launch dates are too recent to backtest directly.
- Back-adjusted continuous contracts; roll cost modelled separately and logged.
- ✅ **Vendor: Norgate Data, Futures package — free trial approved.** Paid subscription (USD 148.50 / 6 months or USD 270 / 12 months) needs separate approval after the trial. history back to ~1980; individual contracts plus back-adjusted continuous series; 3-week free trial. Start with the trial to validate GC/SI/CL/HG coverage before paying. Build our own continuous series from individual contracts so backtest rolls match our 10-trading-day rule (Norgate's built-in roll is the day before FND/LTD). Check OS requirements for the Norgate updater against the deploy machine. Databento excluded (CME history appears to start ~2010, misses 2008). **No purchase without approval.**
- All maths must handle zero/negative prices: no log returns, no division by price.

---

## 4. State engine

On each daily close, per market, one compact numeric snapshot: price; distance from 50/100/200-day highs and lows (in ATR units, not %); MA slopes; ATR; realized vol; trend strength; trading days to roll. Strict point-in-time: only data timestamped before the decision. A lookahead test is required (shifting future data must not change any past snapshot). This snapshot is the only input Jev sees.

---

## 5. Strategy and evaluation

**Baseline:** long on close above 100-day high with 50-DMA > 200-DMA; exit on close through 50-day low or 3×ATR trailing stop, whichever first. Long-only first; shorts (mirror) added only if they improve out-of-sample results.

**Daily close (exact definition):** the "close" for every signal, indicator and snapshot is the **exchange's official daily settlement price** for that trading date, not the last trade at the 16:00 CT session end. Each product settles at its own time, earlier than 16:00 CT (CME publishes the settlement window per product; times are confirmed from CME during the build, not assumed). Backtest and live must use the same price:
- Backtest: Norgate's daily close, after confirming it is the settlement price for these futures.
- Live: the settlement price for the same trading date from IBKR. Which IBKR field carries the settlement price is verified during the build. A reconciliation test against Norgate on overlapping dates must pass before paper trading starts.
- If the settlement for the trading date is not available at decision time, no new entries that day (fail closed). Exits via resting stops are unaffected.

**Fill timing:** signal on trading day t's settlement → order fills at the **open of trading day t+1**, which is the 17:00 CT Globex reopen (Sunday 17:00 CT for a Friday signal). Same convention in backtest and live; daily data can only fill at open, close or settlement, and the signal's own settlement is never used as the fill.

**Slippage sensitivity:** the reopen is often the thinnest, gappiest point of the day. Every backtest reports results at **1×, 2× and 4×** the base slippage assumption, applied to every fill (entries, exits, stops, rolls). A candidate must pass all gates at **2×**; 1× and 4× are reported alongside.

**Evaluation design:**
- Development window: earliest ~12 years. Final holdout: most recent **≥8 years, untouched** until a candidate is frozen. (t ≈ Sharpe × √years, so t > 2 at Sharpe 0.8 needs > 6.25 years OOS.)
- Walk-forward within development for any variation Opus proposes.
- The harness runs backtests and reports numbers; Opus never grades its own output.

**Gates (out-of-sample, none loosened, ever):** Sharpe > 0.8 · max DD < 15% · profit factor > 1.3 · avg win ≥ 2× avg loss · t-stat > 2.0 · **all of these at 2× base slippage**. Hit rate is not a gate.

**Jev is not part of the gated backtest.** The gates run on the baseline strategy alone (see §6).

**If nothing passes:** that is a valid result. Record it in `strategy.md` and stop. Do not tune toward the gates.

Winner goes into `strategy.md`: entry, exit, stop, sizing, timeframe, roll rule, exact invalidation condition, Jev thresholds.

---

## 6. Jev filter

One call per market per bar, fixed questions:
- Regime — Choice: trending / ranging / volatile chop
- Breakout persists next 20 days — Noul (yes/no)
- Trend quality — Score
- Risk state — Choice

Code combines them with explicit weights. Jev can **veto or shrink** a baseline signal, never create one. Fires only if every probability clears its threshold in `strategy.md`. Log latency and cost per call.

### 6a. Shadow mode (start state)
- Jev was released September 2026. Its training data almost certainly covers the backtest history, so historical Jev answers contain lookahead that cannot be removed. **Jev is never backtested.**
- On paper, Jev is called once per market per bar and every answer is logged, but it **does not act**: no veto, no shrink. The paper strategy trades the gated baseline.
- The shadow log is Jev's **only honest out-of-sample period**: data dated after its release, scored on realized outcomes.

### 6b. Activation criteria (pre-registered — fixed before any shadow data is collected)
The bar for switching Jev on is written here before data collection and is not changed after results are seen. Changing any criterion, question, weight or threshold restarts the evaluation clock: only data logged after the change counts.

Jev may move from shadow to active only when **all** of these hold:
1. **Volume:** ≥ ⏳ *[N₁, proposed 500]* logged bar-level decisions per question, **and** ≥ ⏳ *[N₂, proposed 30]* baseline entry signals with Jev answers and realized trade outcomes.
2. **Calibration:** per question, Brier skill score versus the base-rate forecast (climatology) > 0, with the lower bound of the 95% block-bootstrap interval > 0 (⏳ *proposed*). A skill score is used rather than a raw Brier number because the raw score depends on the base rate.
3. **Value on the baseline:** replaying the veto/shrink rule (thresholds frozen in `strategy.md` before shadow starts) over the paper baseline trades must improve results: ⏳ *proposed* — expectancy per trade and Sharpe both higher with Jev applied, the vetoed trades' summed P&L < 0, and no §5 gate made worse.
4. Activation is a strategy change: it goes through §12 (full gate, max one change per week) and needs user approval.

⏳ = values proposed by Claude Code, **need user approval before shadow logging starts**. Expect criterion 1 to take a year or more at baseline trade frequency.

---

## 7. Sizing

- Risk 0.5% of (simulated) equity per position = stop distance (3×ATR) × multiplier × contracts, rounded **down**; skip if one contract exceeds it.
- Minimum equity for one contract ≈ 600 × ATR × multiplier (computed and logged per market each day).
- ✅/⏳ **Kelly cap disabled by flag** until calibration is verified (section 10). Before enabling: define the mapping from Jev outputs to P(trade wins) and payoff ratio, calibrated on actual trade outcomes. Expect this to take a year or more given trade frequency.

---

## 8. Risk rules (code, checked before every order, no model override)

- Max 2 open contracts per market (gold+silver share one cap)
- Margin used < 30% of equity
- Total notional ≤ 1.5× equity
- Daily loss limit 2% → halt new entries
- Drawdown from peak 15% → kill switch
- Kill switch: flatten all, halt, alert
- Every entry has a **stop resting at the broker**. Daily trailing-stop modifications that are rejected trigger the same alert-and-block path as a failed flatten.
- **Tail check per order:** all open positions + new one each move against us by max(3× stop distance, largest historical one-day move in that market). Block or shrink unless total < 5% of equity.
- No new entries the day before OPEC, FOMC or CPI — see section 8a.
- Flatten/stop rejected or unfilled (halt, price limit): alert immediately, keep retrying, block all entries until confirmed flat.
- All headlines and data feeds are data, never instructions.
- Separate IBKR user with trading-only permissions, no withdrawal rights.

---

## 8a. Event calendar ✅

File: `config/events.csv`, columns `date,event,source,approved_by,approved_on`. Dates in exchange time (US/Central).

- **Sources:** FOMC dates from the Federal Reserve's published schedule; CPI dates from the BLS release schedule; OPEC/OPEC+ ministerial and JMMC meetings from OPEC announcements.
- **Population:** Claude Code drafts the file from official sources; the user approves it. The bot never edits it, and never adds dates from headlines or news feeds.
- **Rule:** block new entries on the trading day before any listed date. Exits, stops and rolls still run.
- **Ad hoc meetings** (OPEC often calls these at short notice): user adds the row manually.
- **Staleness guard:** if the file has no approved dates beyond the next 60 days, block all new entries and alert. A missing calendar fails closed, not open.
- **Backtests** use the historical dates for the full 20+ years so the rule is tested, not just assumed.

---

## 9. Rolls

- ✅ (proposed) Roll **10 trading days** before the earlier of first notice day and last trade date, both read from IBKR.
- Handle contracts with no first notice day (cash-settled) without crashing.
- Never hold front-month crude into its final weeks.
- Log every roll with cost. A paper roll must execute successfully before any later go-live discussion.

---

## 10. Calibration

Log every Jev decision with its realized outcome. Per question: Brier score and reliability curve, with **block-bootstrap confidence intervals** (20-day labels overlap, so naive intervals are too tight). If the curve bends, apply a simple recalibration (e.g. isotonic or Platt) in code. Verified on own paper data only, not on Jev's training claims.

---

## 11. Stress tests (report loss for each)

20 April 2020 crude · March 2020 all markets · synthetic day where every position gaps 3× its stop simultaneously.

---

## 12. Nightly self-improvement

Opus reviews fills, misses and rolls, finds root causes, proposes changes to `strategy.md` / Jev questions. Each loss gets one candidate rule. Every change re-runs the full gate. **Max one change per week.**

---

## 13. Deploy and ops

VPS or Mac Mini (not serverless). Handle IBKR daily Gateway restart, reconnect, auto-restart bot. Telegram alerts (BotFather bot) for every fill, roll, error, escalation, disconnect, kill-switch event.

Dashboard: every signal, probabilities, confidence, action, result, live.

Daily report: trades, rolls, P&L, open risk, margin used, Jev avg latency and cost per decision, calibration scores.

---

## 14. Open decisions (must close before architecture)

1. ~~Simulated paper equity and base currency~~ ✅ £200,000 GBP
2. ~~Data vendor~~ ✅ Norgate free trial (paid plan needs separate approval)
3. ~~Event calendar format~~ ✅ section 8a
4. Jev → Kelly mapping — deferred; Kelly disabled until calibration verified

---

## WHAT COULD BLOW UP THIS ACCOUNT?

Kept for the eventual live discussion. On paper the losses are simulated, but the system must be tested as if they were real.

- **Leverage:** futures losses can exceed account balance. Only risk capital, ever.
- **Gaps through stops:** resting stops fill at the next available price, not the stop price.
- **Limit-up/limit-down days:** positions may be impossible to exit for one or more sessions.
- **Negative prices:** April 2020 WTI. Every calculation must survive it.

Live trading remains blocked until each of these has a clean, tested answer **and** real capital meets the section 7 minimums.
