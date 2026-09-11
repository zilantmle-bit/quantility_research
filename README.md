Ниже — версия README, которую я бы использовал как основу реального quant-research проекта: без маркетинговых формулировок, с чётким разделением **signal predictability → latency → execution → economic significance**.

 # HFT Signal Decay & Execution Economics in BTC/USDT Perpetuals

 ## 1\. Research Objective

 This project investigates whether short-horizon L2/order-flow information in the Binance BTCUSDT perpetual market can be converted into **positive execution-adjusted economic edge** after accounting for latency, fees, spread, market impact, slippage, and maker fill uncertainty.

 The research explicitly separates three questions:

 1. **Statistical predictability** — does the current L2 state predict future price movement?
2. **Latency-adjusted predictability** — does the signal remain informative when execution is delayed?
3. **Economic tradability** — does the remaining edge survive realistic execution costs?

 The central research object is the relationship:

 $$
\text{Signal Strength}
\rightarrow
\text{Predictive Markout}
\rightarrow
\text{Latency Decay}
\rightarrow
\text{Execution}
\rightarrow
\text{Net Economic Edge}
$$

 The objective is not to maximize backtest PnL or build a complex predictive model.

 The objective is to estimate the **latency threshold at which L2 information ceases to be economically exploitable**.

---

 # 2\. Primary Research Hypothesis

 ### H1 — Short-lived predictive information

 L2/order-flow variables contain statistically significant information about future short-horizon mid-price movements.

 ### H2 — Signal decay

 The predictive value of L2 information decays rapidly with prediction horizon.

 ### H3 — Latency decay

 As execution latency increases, the expected markout decreases because the information becomes incorporated into the market before execution.

 ### H4 — Taker economics

 For a taker strategy, statistically significant prediction does not necessarily imply positive net edge because the signal must overcome:

 - half/full spread crossing;
- taker fees;
- market impact;
- slippage;
- latency-induced adverse movement.

 ### H5 — Maker economics

 Maker execution may avoid crossing the spread but introduces a different problem:

 $$
P(\text{fill})
$$

 is endogenous to future order flow.

 Conditional on being filled, adverse selection may materially reduce or eliminate expected markout.

 ### Null hypothesis

 There exists no robust positive execution-adjusted edge after realistic transaction costs and latency.

---

 # 3\. Research Architecture

 The research is intentionally divided into independent layers.

```
Raw Market Data
      │
      ▼
Deterministic L2 Reconstruction
      │
      ▼
Market State at t
      │
      ├── OFI
      ├── Trade Flow
      ├── Queue Imbalance
      ├── Micro-price
      ├── Spread
      └── Depth / Liquidity
      │
      ▼
Future Markout
      │
      ▼
Latency Simulation
      │
      ▼
Execution Model
      │
      ├── Taker
      └── Maker
      │
      ▼
Fees + Slippage + Market Impact
      │
      ▼
Net Economic Edge
```

 This separation is mandatory.

 A signal must first demonstrate predictive value **before** execution assumptions are introduced.

---

 # 4\. Market and Scope

 Primary market:

 **Binance BTCUSDT USDT-M Perpetual**

 The initial research uses one instrument and one venue deliberately.

 This avoids mixing:

 - different matching engines;
- different fee structures;
- different contract specifications;
- different market microstructure;
- cross-exchange latency assumptions.

 Cross-venue generalization is explicitly outside the first research phase.

---

 # 5\. Data Requirements

 ## 5.1 Required market data

 The minimum dataset consists of:

 ### L2 order book

 Incremental depth updates with:

 - exchange timestamp;
- sequence/update identifiers;
- price;
- quantity;
- side;
- additions;
- cancellations;
- quantity modifications.

 The book must be reconstructed deterministically from an initial snapshot and subsequent incremental updates.

 ### Trades

 For each trade:

 - exchange timestamp;
- price;
- quantity;
- trade identifier if available;
- aggressor/side information where available.

 Trades and book updates must be aligned without double-counting executions as independent liquidity changes.

---

 ## 5.2 Timestamps

 At minimum, maintain:

```
exchange_timestamp
local_receive_timestamp
```

 The distinction is critical.

 `exchange_timestamp` represents when the exchange generated/assigned the event.

 `local_receive_timestamp` represents when the information became observable by the research system.

 The latter is the relevant timestamp for information availability.

 If actual trading infrastructure is available, additionally record:

```
order_submission_timestamp
exchange_ack_timestamp
execution_timestamp
```

 These timestamps allow the execution model to be calibrated rather than assumed.

---

 # 6\. Deterministic Order Book Reconstruction

 The research must operate on a reconstructed event stream.

 For every event:

```
event_id
exchange_timestamp
local_receive_timestamp
sequence_number
event_type
side
price
quantity
```

 The reconstruction engine must detect:

 - missing sequence numbers;
- duplicate events;
- out-of-order events;
- reconnects;
- invalid book states.

 Any period containing unrecoverable sequence gaps should be excluded.

 A silently corrupted order book invalidates all downstream OFI and microstructure results.

---

 # 7\. Feature Set

 The first research version intentionally uses a small feature set.

 ## 7.1 Order Flow Imbalance

 OFI should be defined directly from changes in executable bid/ask liquidity.

 Conceptually:

 $$
OFI_t =
\Delta BidLiquidity_t
-
\Delta AskLiquidity_t
$$

 The exact implementation must distinguish:

 - new limit orders;
- cancellations;
- executions;
- price-level changes.

 OFI definitions should not be mixed across event types without explicit accounting.

---

 ## 7.2 Queue Imbalance

 For top-of-book:

 $$
QI_t =
\frac{Q_{bid}-Q_{ask}}
{Q_{bid}+Q_{ask}}
$$

 Multi-level versions may subsequently be tested.

---

 ## 7.3 Micro-price

 For best bid/ask:

 $$
MicroPrice =
\frac{P_{ask}Q_{bid}+P_{bid}Q_{ask}}
{Q_{bid}+Q_{ask}}
$$

 Micro-price is used **only as a predictive feature**.

 It is never used as the future price target.

---

 ## 7.4 Trade Flow

 Signed trade flow captures aggressive buying/selling pressure.

 It must be constructed independently from passive book changes to avoid double-counting.

---

 ## 7.5 Market State Variables

 Additional controls:

 - bid-ask spread;
- top-of-book depth;
- multi-level depth;
- short-term volatility;
- recent traded volume;
- order-book imbalance;
- recent trade intensity.

 The initial feature set should remain intentionally small.

---

 # 8\. Prediction Target

 The first target is **future mid-price markout**, not PnL.

 For horizon $h$:

 $$
M_h =
Mid_{t+h}-Mid_t
$$

 A normalized version can be expressed in basis points:

 $$
M_h^{bps}
=
\frac{Mid_{t+h}-Mid_t}{Mid_t}\times10^4
$$

 Candidate horizons:

```
50ms
100ms
250ms
500ms
1s
2s
5s
```

 The purpose is to estimate the signal's natural decay curve.

---

 # 9\. Why Mid-Price Markout Comes First

 Using PnL as the primary statistical target confounds:

 - prediction;
- spread;
- fees;
- execution;
- latency;
- fill probability.

 This makes it impossible to identify where the edge disappears.

 Therefore:

```
Stage 1:
Signal → Future Markout

Stage 2:
Signal → Latency-adjusted Markout

Stage 3:
Signal → Execution → Net PnL
```

 This decomposition is a central design principle.

---

 # 10\. Latency Model

 Latency must not be represented by one arbitrary fixed number.

 Model:

 $$
L =
L_{network}
+
L_{processing}
+
L_{exchange}
$$

 If empirical measurements are available, use their observed distribution.

 Otherwise, perform sensitivity analysis over a predefined latency grid.

 For each latency $L$:

 $$
t_{effective}=t_{local}+L
$$

 The signal must be evaluated using only information available at $t_{effective}$.

 The primary output is:

 $$
Edge(L)
$$

 or equivalently:

 **Net Economic Edge vs Latency**

 This curve is more informative than a single backtest PnL number.

---

 # 11\. Taker Execution Model

 At simulated execution time $t_{effective}$, the strategy crosses the book using the actual executable L2 liquidity.

 Execution price must account for:

 - available quantity;
- market order size;
- multiple price levels;
- spread;
- market impact;
- slippage.

 The execution price must therefore be derived from the reconstructed order book rather than from mid-price.

 For a buy:

 $$
P_{exec}
=
VWAP(\text{ask liquidity consumed})
$$

 For a sell:

 $$
P_{exec}
=
VWAP(\text{bid liquidity consumed})
$$

 This produces implementation shortfall rather than an artificially optimistic fill.

---

 # 12\. Maker Execution Model

 Maker execution is fundamentally different from taker execution.

 A hypothetical passive order is not automatically filled.

 The model must estimate:

 $$
P(Fill\mid State, Queue, OrderFlow, Lifetime)
$$

 and:

 $$
E[Markout\mid Fill]
$$

 The minimum model should account for:

 - price level;
- queue ahead;
- cancellations ahead;
- executions at the level;
- trades through the level;
- order lifetime;
- partial fills.

 The research must never determine whether an order was filled using future information and then treat the resulting PnL as unconditional strategy performance.

 Fill is an endogenous event.

---

 # 13\. Maker Adverse Selection

 For every actual or simulated fill, calculate post-fill markout:

 $$
AM_h =
Side \times
(Mid_{t+h}-P_{fill})
$$

 where the sign is chosen so that positive values represent favorable movement.

 Evaluate at:

```
100ms
500ms
1s
5s
```

 The critical quantity is:

 $$
E[Markout\mid Fill]
$$

 not simply unconditional future return.

 This directly measures winner's curse/adverse selection.

---

 # 14\. Time Horizons

 Three representations are evaluated.

 ## Clock-time

```
100ms
500ms
1s
5s
```

 Clock-time is essential for actual latency and execution economics.

 ## Event-time

 Examples:

```
10 events
50 events
100 events
```

 Event-time is useful for understanding microstructure dynamics independent of absolute market activity.

 ## Volume-time

 Volume-normalized horizons can be used as a secondary robustness analysis.

 The primary economic analysis remains clock-time because latency and execution occur in physical time.

---

 # 15\. Statistical Models

 The first model should be deliberately simple.

 ## Ridge Regression

 $$
M_h =
\beta_0+
\beta_1 OFI+
\beta_2 QI+
\beta_3 MicroPrice+
\beta_4 TradeFlow+
\beta_5 Spread+
\epsilon
$$

 Ridge is used for:

 - coefficient stability;
- multicollinearity control;
- interpretable baseline;
- OOS robustness.

 No LightGBM/deep learning is required in the first stage.

 A complex model is only justified if the linear baseline demonstrates stable incremental information.

---

 # 16\. Statistical Inference

 Microstructure observations are not IID.

 They exhibit:

 - serial correlation;
- volatility clustering;
- event clustering;
- overlapping labels;
- regime dependence.

 Therefore ordinary IID standard errors are inappropriate.

 Use:

 - HAC/Newey-West style inference where appropriate;
- block bootstrap;
- purged walk-forward validation.

 Bootstrap block length must reflect the relevant dependence structure rather than using arbitrary resampling.

---

 # 17\. Overlapping Labels

 If a prediction at $t$ uses:

 $$
M_{t,1s}
$$

 then nearby observations share future information.

 This creates dependence and invalidates naive statistical inference.

 Possible solutions:

 - non-overlapping observations for primary inference;
- HAC estimators;
- appropriate block bootstrap;
- purged validation.

 The research should report exactly which convention is used.

---

 # 18\. Walk-Forward Validation

 No random train/test split.

 Use chronological walk-forward validation:

```
Train → Test
       →
Train --------→ Test
               →
Train ----------------→ Test
```

 Each test period must represent genuinely unseen future data.

 The purge interval must cover the maximum information overlap created by the label and, for execution experiments, the relevant execution footprint.

---

 # 19\. Multiple Testing

 The research contains multiple:

 - horizons;
- features;
- model specifications;
- regimes;
- latency values.

 Therefore statistical significance must not be interpreted from isolated p-values.

 Use predefined research families and FDR correction where appropriate.

 More importantly, maintain a **frozen primary specification** before examining final OOS results.

 FDR does not eliminate data-snooping risk.

---

 # 20\. Non-Stationarity

 BTC perpetual microstructure changes materially with:

 - volatility;
- liquidity;
- trading activity;
- market regime;
- session;
- large directional moves;
- liquidation events.

 Therefore report results by regime.

 At minimum:

```
low / medium / high volatility
low / high liquidity
normal / stressed market
```

 The objective is not to find the best regime.

 The objective is to determine whether the relationship survives regime changes.

---

 # 21\. Economic Model

 Net economic edge must include:

 $$
NetEdge =
GrossMarkout
-
SpreadCost
-
Fees
-
Slippage
-
MarketImpact
-
LatencyCost
$$

 For maker strategies, replace spread-crossing cost with:

 $$
NetEdge_{maker}
=
ExpectedMarkout
+
MakerRebate
-
AdverseSelection
-
ExecutionCosts
$$

 where execution costs include partial-fill and inventory effects where relevant.

 Funding should be included whenever the holding period makes it economically material.

---

 # 22\. Statistical vs Economic Predictability

 These are explicitly different hypotheses.

 ### Statistical predictability

 $$
E[M_h\mid X_t]\neq0
$$

 does not imply:

 ### Economic predictability

 $$
E[NetPnL\mid X_t]>0
$$

 A signal can be statistically significant but economically useless.

 Example:

```
Expected markout: +0.7 bps
Spread + fees + slippage: 1.1 bps
Net edge: -0.4 bps
```

 Such a signal is statistically informative but not tradable.

---

 # 23\. Primary Research Outputs

 The research should produce five primary plots.

 ### 1\. Signal Decay Curve

 $$
E[M_h\mid Signal]
$$

 versus prediction horizon.

 ### 2\. Latency Decay Curve

 $$
E[M_h\mid Signal,L]
$$

 versus latency.

 ### 3\. Net Edge vs Latency

 Separate curves for:

```
Taker
Maker
```

 ### 4\. Maker Fill/Markout Curve

 $$
P(Fill)
$$

 and

 $$
E[Markout\mid Fill]
$$

 as functions of signal strength and queue conditions.

 ### 5\. OOS Stability

 Coefficient / markout / net-edge stability across chronological test periods and market regimes.

---

 # 24\. Primary Metrics

 The main metrics are:

 - expected markout in bps;
- conditional expected markout;
- net edge per opportunity;
- net PnL after all modeled costs;
- implementation shortfall;
- fill probability;
- adverse-selection markout;
- turnover;
- PnL/turnover;
- OOS stability.

 Sharpe ratio is secondary.

 A high Sharpe from an unrealistic execution model is not evidence of a valid HFT strategy.

---

 # 25\. Leakage Controls

 The following rules are mandatory.

 ### Rule 1

 No feature may use an event with:

 $$
local\_receive\_timestamp > t
$$

 ### Rule 2

 Slow variables are available only after their actual publication/observation timestamp.

 ### Rule 3

 Future order-book events cannot determine whether a hypothetical maker order was initially considered.

 ### Rule 4

 Execution price is determined from the book available at execution time.

 ### Rule 5

 Model selection cannot use final OOS performance.

 ### Rule 6

 Latency distributions used for final evaluation must not be calibrated on the same observations used to report final performance.

---

 # 26\. Research Workflow

```
01. Data validation
        ↓
02. Deterministic L2 reconstruction
        ↓
03. Timestamp / availability validation
        ↓
04. Feature construction
        ↓
05. Unconditional markout analysis
        ↓
06. Signal decay estimation
        ↓
07. OOS linear baseline
        ↓
08. Latency sensitivity
        ↓
09. Taker execution simulation
        ↓
10. Maker queue/fill simulation
        ↓
11. Cost model
        ↓
12. Regime robustness
        ↓
13. Final OOS evaluation
```

 No execution optimization should occur before Stage 5–6 demonstrate genuine predictive information.

---

 # 27\. What Would Constitute Evidence of a Real Result?

 A convincing result requires all of the following:

 1. Positive OOS predictive markout.
2. Clear decay with horizon.
3. Persistence after realistic latency.
4. Stability across independent time periods.
5. Robustness across market regimes.
6. Positive net edge after realistic execution costs.
7. No dependence on one arbitrary parameter choice.
8. No evidence of timestamp or fill leakage.

 A statistically significant coefficient alone is insufficient.

---

 # 28\. What Would Falsify the Hypothesis?

 The research should be considered negative if:

 - signal significance disappears OOS;
- markout disappears after realistic latency;
- edge disappears after fees/slippage;
- maker profitability depends on unrealistic fills;
- results exist only in one market regime;
- small changes in latency assumptions eliminate profitability;
- results depend heavily on one feature/horizon combination.

 A negative result is valid research.

---

 # 29\. Expected Contribution

 The intended contribution is not another demonstration that OFI predicts short-term BTC returns.

 The more interesting result is an empirical estimate of the **information-to-execution conversion frontier**:

 $$
\boxed{
Signal
\rightarrow
Decay
\rightarrow
Latency
\rightarrow
Execution
\rightarrow
Economic\ Edge
}
$$

 The key research question becomes:

 > **How much of short-lived L2 predictive information remains economically exploitable at a given latency, and does the answer differ fundamentally between taker and maker execution?**

---

 # 30\. Research Integrity

 The project prioritizes:

 - causal timestamp ordering;
- deterministic market replay;
- realistic execution;
- conservative assumptions;
- chronological OOS validation;
- transparent negative results.

 The objective is not to produce a profitable backtest.

 The objective is to determine whether a profitable mechanism survives contact with the actual microstructure of BTCUSDT perpetuals.

---

 # 31\. Final Research Standard

 The project is considered **research-grade** only if the final conclusion can be stated in terms of:

 $$
\boxed{
E[NetEdge\mid Signal,\ Latency,\ Execution,\ Regime]
}
$$

 with all four dimensions evaluated out-of-sample.

 The final deliverable should therefore answer one precise question:

 > **At what latency, execution regime, and market state does L2 information in BTCUSDT perpetuals stop being economically exploitable?**
