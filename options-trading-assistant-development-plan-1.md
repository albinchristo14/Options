# Options Trading Assistant — Development Plan

## 1. Project Overview

Build a **local-first Indian options trading assistant** that connects to Groww for market data and, after explicit user approval, can place an approved options order through the Groww API.

The system is **not an autonomous trading bot**.

Core workflow:

```text
Groww Live Data
      ↓
Market Context
      ↓
Options Scanner
      ↓
Strategy / Setup Detection
      ↓
Trade Construction
      ↓
Risk Validation
      ↓
Trade Proposal
      ↓
USER REVIEW
  ├── Reject
  ├── Modify
  └── Approve
          ↓
Final Live Validation
          ↓
Groww Order API
```

The user remains the final decision-maker for every live trade.

---

# 2. Initial Scope

## Instruments

V1 should support:

- NIFTY options
- BANKNIFTY options

Future expansion may include:

- FINNIFTY
- SENSEX
- Selected liquid stock options

## Trading style

- Intraday options buying
- CE and PE
- No naked option selling in V1
- No automatic trading
- One active live position at a time initially
- User approval required before every live order

## Initial timeframes

- 15-minute: market regime
- 5-minute: setup detection
- 1-minute: entry confirmation
- Tick/WebSocket data: live validation and execution-price checks

---

# 3. Primary Data Provider

Use **Groww Trading API** as the primary market-data and execution provider.

The implementation should verify the current Groww API documentation during development and account for API limits, authentication, WebSocket behaviour, instrument-token discovery, historical-data availability, and order restrictions.

Required data capabilities:

- Live index quotes
- Live option quotes
- Option chain
- LTP
- OHLC
- Volume
- Open Interest
- Change in OI
- IV
- Greeks
- Bid/ask / market depth where available
- Historical candles
- Historical F&O/OI data where available
- WebSocket streaming
- Order placement after explicit approval

Do not depend on a second provider in V1 unless a required Groww data field is unavailable or unreliable.

---

# 4. Local-First Architecture

The application should run primarily on the user's PC.

Recommended architecture:

```text
                         GROWW
                           │
              ┌────────────┴────────────┐
              │                         │
        WebSocket Feed             REST APIs
              │                         │
              └────────────┬────────────┘
                           ↓
                    DATA INGESTION
                           ↓
                    LOCAL DATABASE
                           ↓
                  MARKET DATA ENGINE
                           ↓
                   STRATEGY ENGINE
                           ↓
                  OPPORTUNITY ENGINE
                           ↓
                    RISK ENGINE
                           ↓
                  TRADE PROPOSAL
                           ↓
                         USER
                           ↓
                     APPROVE/REJECT
                           ↓
                 FINAL LIVE VALIDATION
                           ↓
                     GROWW ORDER API
```

Suggested technology:

- Python 3.12+
- FastAPI for local backend/API
- WebSocket support
- PostgreSQL or TimescaleDB for persistent data
- Redis optional for fast in-memory state
- React/Next.js or another modern frontend for the local dashboard
- Pandas/NumPy for analysis
- Pydantic for data validation
- AsyncIO for live data processing
- Docker optional

Keep strategy logic independent of the UI and Groww API.

---

# 5. Data Model

The system should store raw and derived market information.

## Underlying data

For NIFTY/BANKNIFTY:

- Timestamp
- Instrument
- LTP
- Open
- High
- Low
- Close
- Volume where available
- VWAP
- EMA 9
- EMA 20
- EMA 21
- EMA 50
- EMA 200
- RSI
- ATR
- Swing highs/lows
- Support/resistance
- Market regime

## Option contract data

For each relevant contract:

- Instrument token
- Symbol
- Underlying
- Expiry
- Strike
- CE/PE
- LTP
- Open
- High
- Low
- Close
- Volume
- OI
- Change in OI
- IV
- Delta
- Gamma
- Theta
- Vega
- Rho
- Bid
- Ask
- Spread
- Timestamp

## Derived option data

Calculate locally:

- Relative volume
- OI buildup
- OI unwinding
- Long buildup
- Short buildup
- Short covering
- Long unwinding
- PCR
- Strike-wise OI concentration
- OI migration
- IV change
- IV percentile when enough history exists
- Premium momentum
- Liquidity score
- Contract quality score

---

# 6. Instrument Selection

Do not subscribe to every available option.

The system should dynamically determine the ATM strike.

Example:

```text
NIFTY spot = 25,180
ATM ≈ 25,200
```

Monitor a configurable range around ATM.

Initial range:

```text
ATM - 2 strikes
ATM - 1 strike
ATM
ATM + 1 strike
ATM + 2 strikes
```

The exact number of strikes should be configurable.

Expand the range when required for:

- Wide support/resistance areas
- Large OI concentrations
- Expiry-specific conditions
- Contract-selection analysis

---

# 7. Market Regime Engine

Before searching for an option trade, classify the underlying.

Possible states:

```text
STRONG_BULLISH
BULLISH
NEUTRAL
BEARISH
STRONG_BEARISH
CHOPPY
```

Do not use a single indicator to classify the regime.

## 15-minute inputs

- EMA 20
- EMA 50
- VWAP
- Swing structure
- ATR
- RSI

## 5-minute inputs

- EMA 9
- EMA 21
- VWAP
- RSI
- ATR
- Volume
- Price structure

## Initial bullish hypothesis

Most of the following should align:

- 15m close above EMA20
- EMA20 above EMA50
- 5m price above VWAP
- 5m EMA9 above EMA21
- Higher highs / higher lows
- Positive momentum

## Initial bearish hypothesis

Opposite structure:

- 15m close below EMA20
- EMA20 below EMA50
- 5m price below VWAP
- 5m EMA9 below EMA21
- Lower highs / lower lows
- Negative momentum

## Choppy regime

Potential indicators:

- Repeated VWAP crossings
- Frequent EMA9/EMA21 crossings
- Overlapping candles
- Failed breakouts
- Weak relative volume
- Compressed ATR

Choppy conditions should normally block directional option trades.

All regime thresholds are initial hypotheses and must be validated by backtesting.

---

# 8. Trading Session Rules

Initial observation window:

```text
09:15–09:25
```

Do not normally generate live trades during the opening observation period.

Initial operating windows:

```text
09:25–10:30  Primary
10:30–12:00  Secondary
12:00–13:30  Conservative
13:30–14:45  Secondary
14:45–15:15  Special/expiry-aware logic
```

These windows should be configurable and statistically evaluated.

Do not assume a time period is profitable until the backtest confirms it.

---

# 9. Strategy 1 — Breakout Continuation

This is the first strategy to implement.

## Concept

Detect a meaningful support/resistance breakout with confirmation from:

- Price structure
- Volume
- VWAP
- Momentum
- Higher timeframe
- Option activity
- Liquidity

## Initial conditions

### Structural breakout

5-minute candle closes beyond a meaningful resistance/support level.

### Volume

Initial hypothesis:

```text
Breakout candle volume >= 1.3 × median volume of previous 20 candles
```

Use median rather than simple average initially.

### Bullish breakout

Require:

- Breakout above resistance
- Price above VWAP
- 5m momentum positive
- 15m structure not strongly bearish
- CE contract confirmation
- Acceptable liquidity
- Acceptable risk/reward

### Bearish breakdown

Mirror the conditions:

- Breakdown below support
- Price below VWAP
- 5m momentum negative
- 15m structure not strongly bullish
- PE confirmation
- Acceptable liquidity
- Acceptable risk/reward

---

# 10. Breakout Retest

Support both:

## Retest entry

```text
Resistance
──────────────
       ↑ breakout
       │
       ↓ retest
       │
       ↑ confirmation
```

The system should identify:

- Breakout
- Return toward breakout level
- Failure to decisively break back through
- Confirmation candle
- Valid option entry

## Momentum entry

If price does not retest and continues strongly, a separate momentum-breakout mode may be allowed.

Backtesting must compare:

- Immediate confirmed breakout
- Retest entry
- Momentum continuation

Do not assume one is superior.

---

# 11. Strategy 2 — Trend Pullback

## Bullish setup

Requirements:

- 15m bullish regime
- Price above VWAP
- 5m pullback toward EMA21/VWAP
- No major structural breakdown
- Selling volume decreases during pullback
- Bullish reversal/confirmation candle
- CE option confirmation
- Acceptable liquidity
- Acceptable R:R

## Bearish setup

Mirror conditions:

- 15m bearish regime
- Price below VWAP
- Pullback toward EMA21/VWAP
- No structural recovery
- Selling/buying volume behaviour confirms
- Bearish confirmation
- PE confirmation

The objective is to avoid buying a fully extended candle and instead identify controlled pullbacks.

---

# 12. Strategy 3 — Breakdown Continuation

Mirror the breakout strategy for bearish conditions.

Initial conditions:

- Meaningful support identified
- 5m candle closes below support
- Relative volume expansion
- Price below VWAP
- EMA9 below EMA21
- 15m structure supports bearish direction
- PE activity confirms
- Contract liquidity acceptable
- R:R acceptable

Avoid entries after an excessively extended move.

---

# 13. Strategy 4 — Reversal

This should have the strictest filters.

Do not use:

```text
RSI < 30 → CALL
RSI > 70 → PUT
```

as a standalone strategy.

## Bullish reversal hypothesis

Potential sequence:

```text
Major support
      ↓
Break/sweep below support
      ↓
Rejection
      ↓
Reclaim support
      ↓
Momentum reversal
      ↓
Volume confirmation
      ↓
Option confirmation
```

Potential requirements:

- Meaningful support
- Rejection/sweep
- Reclaim
- 1m/5m momentum reversal
- Acceptable VWAP relationship
- Volume confirmation
- CE confirmation
- Strong enough R:R

Bearish reversal is the mirror image.

Counter-trend reversals should receive stricter scoring and lower priority than aligned trend setups unless backtesting demonstrates otherwise.

---

# 14. Option Contract Selection Engine

When a directional setup appears, do not automatically select ATM.

Evaluate nearby contracts.

Initial candidate universe:

```text
ATM - 2
ATM - 1
ATM
ATM + 1
ATM + 2
```

For directional option buying, use an initial preferred delta hypothesis of approximately:

```text
0.40–0.65
```

This is not an assumption that this range is optimal. Test alternatives during validation.

## Contract score factors

### Liquidity

- Volume
- OI
- Bid/ask spread

### Responsiveness

- Delta
- Gamma

### Premium characteristics

- Momentum
- IV
- IV change

### Risk

- Premium price
- Stop distance
- Expected reward
- R:R

The system should select the contract with the best combination, not simply the closest-to-ATM strike.

---

# 15. Liquidity Rules

A contract should be rejected when liquidity is inadequate.

Initial spread hypothesis:

```text
Bid/ask spread <= 1.5% of mid-price
```

Also consider:

- Minimum OI
- Minimum volume
- Market depth where available
- Recent trade frequency

Thresholds must be configurable by underlying, expiry and option price.

Do not use one absolute rupee spread threshold for all contracts.

---

# 16. IV Engine

Monitor:

- Current IV
- IV change
- IV percentile once enough history exists
- Option premium change
- Underlying price change

The scanner should recognize:

```text
Underlying signal is bullish
BUT
Option IV is abnormally elevated
```

and potentially downgrade or reject the contract.

Do not treat IV alone as a directional signal.

---

# 17. OI Analysis

OI is a confirmation layer.

Classify:

### Long buildup

```text
Price ↑
OI ↑
```

### Short buildup

```text
Price ↓
OI ↑
```

### Short covering

```text
Price ↑
OI ↓
```

### Long unwinding

```text
Price ↓
OI ↓
```

Do not interpret these classifications as guaranteed future direction.

Combine OI with:

- Underlying structure
- Price
- Volume
- VWAP
- Option premium
- Market regime

---

# 18. Option-Chain Structure

Calculate:

- Total PCR
- Relevant-strike PCR
- Call OI concentration
- Put OI concentration
- Change in OI
- OI migration
- Strike-wise volume
- Unusual activity

Example:

```text
25,100 PE OI ↓
25,200 PE OI ↑
```

This may indicate a change in where put positioning is concentrated.

Treat it as contextual evidence rather than a deterministic support prediction.

---

# 19. Unusual Activity Detector

For each option:

```text
Relative Volume =
Current volume / expected volume for current time of day
```

Flag abnormal activity.

Example:

```text
Relative volume = 3.4x
```

This should trigger attention but should not independently create a trade.

---

# 20. Signal Scoring

Each qualified setup receives an internal evidence score.

Initial framework:

```text
Market regime            20
Price structure          20
Momentum                 15
Volume                   15
Options confirmation     15
Liquidity                 5
Risk/reward              10
                         ---
                         100
```

Example:

```text
Market regime             18/20
Price structure            17/20
Momentum                   14/15
Volume                     13/15
Options confirmation       12/15
Liquidity                   5/5
Risk/reward                 8/10
                           ----
                            87/100
```

Initial signal states:

```text
< 60       NO TRADE
60–69      WATCH
70–79      QUALIFIED
80–89      HIGH-CONFIDENCE SETUP
90–100     EXCEPTIONAL SETUP
```

These are internal setup-strength categories, NOT win probabilities.

The system must later measure whether scores correlate with actual outcomes.

---

# 21. Hard Rejection Rules

A setup should be rejected regardless of score if:

- Market data is stale
- Contract is illiquid
- Bid/ask spread is excessive
- No valid invalidation/stop exists
- R:R is below minimum
- Underlying and option signals strongly conflict
- Price has already moved beyond the entry zone
- Market is in detected chop
- Duplicate position/setup exists
- Contract/expiry is invalid
- Major event/expiry conditions violate configured rules
- Price has gapped beyond the planned entry
- Execution price materially destroys expected R:R

Hard filters must take priority over the score.

---

# 22. Entry Zone

Do not define the trade as one exact price.

Example:

```text
Ideal entry:       ₹142
Acceptable range: ₹140–₹145
Invalid above:     ₹150
```

If price moves outside the valid range before approval:

```text
SIGNAL INVALIDATED
```

Do not allow the user to approve an obviously stale trade proposal.

---

# 23. Stop-Loss Model

Primary stop should be derived from the **underlying market structure**.

Example:

```text
Breakout = 25,200
Underlying invalidation = 25,155
```

The option's expected risk is then calculated from the underlying move and current option characteristics.

Also implement a maximum premium-loss safeguard.

Avoid relying solely on:

```text
Option price -20% = stop
```

because option premium behaviour changes with:

- Delta
- IV
- Time decay
- Gamma

---

# 24. Target Model

Initial target methodology:

### Target 1

Nearest meaningful underlying resistance/support.

### Target 2

Next structural level.

Estimate the corresponding option premium using current option characteristics.

Example:

```text
Underlying:
25,200 → 25,280

Estimated option:
₹142 → ₹175–185
```

Clearly label option targets as estimates.

Do not represent them as guaranteed prices.

---

# 25. Risk/Reward

Initial hard minimum:

```text
R:R >= 1:1.5
```

Preferred:

```text
R:R >= 1:2
```

Test these thresholds statistically.

The system must show:

- Entry
- Stop
- Target
- Risk per unit
- Total planned risk
- Potential reward
- R:R

---

# 26. Position Sizing

Position sizing must be based on configured account risk.

User configuration:

```text
Account capital
Maximum risk per trade %
Maximum quantity
Maximum daily loss
Maximum open positions
```

The engine calculates:

```text
Maximum allowed loss
÷
Risk per contract
=
Maximum quantity
```

Then round to the valid lot size.

The user can modify the quantity before approval.

The final order validator must recalculate risk after any user modification.

---

# 27. Trade Proposal UI

Example:

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
       OPPORTUNITY FOUND
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

NIFTY
Bullish Breakout Continuation

OPTION
NIFTY 25,200 CE
Expiry: XX-XX-2026

SIGNAL
84 / 100

ENTRY
₹142 – ₹146

STOP
Underlying: 25,155

TARGET
₹180 / ₹195

R:R
1 : 2.1

SUGGESTED QTY
75

MAX PLANNED LOSS
₹1,800

WHY?

✓ 15m bullish structure
✓ 5m resistance breakout
✓ Price above VWAP
✓ Volume 1.7× median
✓ EMA confirmation
✓ CE activity increasing
✓ Liquidity acceptable
✓ IV acceptable

RISKS

⚠ Market approaching resistance

[ REJECT ]
[ MODIFY ]
[ APPROVE TRADE ]
```

The UI must show enough evidence that the user can independently inspect the setup.

---

# 28. Approval Workflow

Approval must be explicit.

```text
Opportunity detected
        ↓
User opens proposal
        ↓
User reviews
        ↓
User can modify quantity
        ↓
User presses APPROVE
        ↓
System performs final live checks
        ↓
Order placed only if all checks pass
```

No background order placement.

No automatic order retries.

No automatic strategy override.

---

# 29. Final Pre-Order Validation

Immediately before submitting an order:

```text
✓ Groww connection healthy
✓ Market data fresh
✓ Instrument valid
✓ Expiry valid
✓ Contract still exists
✓ Current LTP checked
✓ Current bid/ask checked
✓ Spread acceptable
✓ Signal still active
✓ Underlying still within setup conditions
✓ Entry still within allowed zone
✓ Stop still valid
✓ R:R still acceptable
✓ Quantity valid
✓ Lot size valid
✓ Risk limit not exceeded
✓ No duplicate order
✓ No conflicting active position
```

If any critical check fails:

```text
ORDER BLOCKED
Reason: <specific reason>
```

Do not automatically place the order.

---

# 30. Groww Order Execution

Only after explicit user approval and final validation:

```text
Trade Proposal
      ↓
Approval
      ↓
Final Validation
      ↓
Groww Order API
```

Store:

- Order request
- Timestamp
- Instrument
- Quantity
- Price/order type
- Groww order ID
- API response
- Final execution price
- Slippage
- Status

Never expose API secrets in frontend code.

Store Groww credentials securely in local environment/secret storage.

---

# 31. Initial Execution Policy

V1 should support:

- Manual approval
- Market/limit order depending on configured strategy
- One order attempt
- Explicit error reporting

Do not implement automatic retry loops initially.

If an order fails:

```text
ORDER FAILED
Reason: ...
```

User decides what to do next.

---

# 32. Post-Trade Monitoring

After execution:

```text
Entry
  ↓
Position monitor
  ↓
Target / stop / invalidation
  ↓
Alert
  ↓
User decision
```

Initial version should keep exits user-approved as well.

The system can alert:

```text
TARGET ZONE REACHED
```

or:

```text
STOP / INVALIDATION REACHED
```

but should not automatically exit in V1.

---

# 33. Trade Journal

Record every signal, not just executed trades.

Each opportunity should contain:

- Timestamp
- Underlying
- Direction
- Strategy
- Signal score
- Market regime
- Strike
- Expiry
- Entry
- Stop
- Target
- Suggested quantity
- User action
- Execution result
- Maximum favourable excursion
- Maximum adverse excursion
- Final outcome
- R result

Statuses:

```text
WATCH
QUALIFIED
APPROVED
REJECTED
INVALIDATED
EXECUTED
CLOSED
EXPIRED
```

---

# 34. Paper Trading

Before real execution:

```text
BACKTEST
   ↓
HISTORICAL REPLAY
   ↓
LIVE PAPER TRADING
   ↓
MANUAL APPROVAL + REAL EXECUTION
```

The paper-trading engine must use the same signal engine as live trading.

No separate "fake strategy."

---

# 35. Backtesting Engine

The backtester must prevent look-ahead bias.

At every simulated timestamp, the strategy may only access information that would actually have been available at that time.

Example:

```text
09:30
→ Analyze data available at 09:30

09:31
→ Receive new data

09:32
→ Evaluate setup

09:33
→ Simulate entry

09:34+
→ Track outcome
```

Never allow future candles, future OI, future IV, or future option prices to influence an earlier decision.

---

# 36. Backtest Metrics

Record:

- Total signals
- Trades
- Win rate
- Average win
- Average loss
- Average R
- Expectancy
- Profit factor
- Maximum drawdown
- Maximum consecutive losses
- Average holding time
- MFE
- MAE
- Slippage sensitivity
- Time-of-day performance
- Strategy performance
- NIFTY vs BANKNIFTY
- CE vs PE
- Expiry distance
- Delta range
- Market-regime performance

Do not optimize solely for win rate.

---

# 37. Out-of-Sample Validation

Split historical data.

Example:

```text
Development period
        ↓
Strategy design
        ↓
Validation period
        ↓
Final untouched evaluation
```

After initial validation, use walk-forward testing.

Do not repeatedly tune rules against the same final dataset.

---

# 38. Strategy Parameter Configuration

All important thresholds must be configurable.

Example:

```yaml
strategy:
  min_rr: 1.5
  preferred_rr: 2.0
  breakout_volume_multiplier: 1.3
  preferred_delta_min: 0.40
  preferred_delta_max: 0.65
  max_spread_percent: 1.5
  atm_strikes_each_side: 2
  minimum_signal_score: 70
```

The configuration should be versioned.

Every backtest and live signal should record the strategy configuration version used.

---

# 39. AI Layer

AI should not independently generate or override trading decisions.

Use AI for:

- Explaining signals
- Summarizing market context
- Explaining why a setup passed/failed
- Summarizing trade history
- Natural-language querying
- Reviewing journal statistics

Example:

> "Why did this NIFTY call opportunity trigger?"

AI receives structured facts from the strategy engine and explains them.

The quantitative engine remains authoritative.

---

# 40. Alerts

Support:

## Local desktop notification

Example:

```text
NIFTY opportunity detected
25,200 CE
Score: 84
R:R: 1:2.1
Open dashboard to review.
```

## Telegram

Same information can be sent to a private Telegram channel/chat.

Telegram alerts must not directly execute trades.

Approval should happen in the local application.

---

# 41. Dashboard

Recommended layout:

## Market panel

```text
NIFTY       25,XXX
BANKNIFTY   XX,XXX

Market regime:
BULLISH

VWAP:
25,XXX

Support:
25,XXX

Resistance:
25,XXX
```

## Opportunity panel

```text
NIFTY 25,200 CE

Breakout continuation

84 / 100

Entry: 142–146
Stop: 25,155 underlying
Target: 180 / 195
R:R: 1:2.1
Qty: 75

[VIEW ANALYSIS]
[REJECT]
[MODIFY]
[APPROVE]
```

## Option-chain panel

Show:

- Strike
- CE LTP
- CE OI
- CE ΔOI
- CE Volume
- CE IV
- PE LTP
- PE OI
- PE ΔOI
- PE Volume
- PE IV

Highlight the selected contract.

## Active trade panel

Show:

- Entry
- Current price
- P/L
- Stop
- Target
- Time in trade
- Underlying state
- Exit/invalidation alerts

---

# 42. Database Tables

Suggested initial schema:

## instruments

- id
- exchange
- symbol
- underlying
- instrument_token
- instrument_type
- strike
- expiry
- option_type
- lot_size

## market_ticks

- timestamp
- instrument_token
- ltp
- bid
- ask
- volume
- oi

## candles

- timestamp
- instrument
- timeframe
- open
- high
- low
- close
- volume
- oi

## option_snapshots

- timestamp
- underlying
- expiry
- strike
- option_type
- ltp
- volume
- oi
- delta
- gamma
- theta
- vega
- rho
- iv
- bid
- ask

## signals

- id
- timestamp
- underlying
- strategy
- direction
- score
- state
- reason_json
- configuration_version

## trade_proposals

- signal_id
- contract
- entry_low
- entry_high
- stop
- target_1
- target_2
- suggested_quantity
- planned_risk
- rr
- created_at
- invalidated_at

## orders

- proposal_id
- groww_order_id
- status
- order_type
- quantity
- requested_price
- executed_price
- timestamp
- error

## trades

- order_id
- entry
- exit
- quantity
- pnl
- r_multiple
- mfe
- mae
- duration
- exit_reason

---

# 43. Logging

Every important event must be logged.

Examples:

```text
DATA_CONNECTED
DATA_DISCONNECTED
DATA_STALE
SIGNAL_CREATED
SIGNAL_INVALIDATED
USER_APPROVED
USER_REJECTED
FINAL_VALIDATION_FAILED
ORDER_SUBMITTED
ORDER_FILLED
ORDER_FAILED
POSITION_UPDATED
TRADE_CLOSED
```

Logs must include timestamps and enough context to reproduce the decision.

---

# 44. Safety Controls

Implement hard limits:

- Maximum risk per trade
- Maximum daily loss
- Maximum number of trades/day
- Maximum open positions
- Maximum quantity
- Trading session window
- Expiry-day restrictions
- Stale-data block
- Duplicate-order block
- Duplicate-position block
- API connection-health block

These controls must live in the execution/risk layer and cannot be overridden by AI.

---

# 45. Failure Handling

If Groww WebSocket disconnects:

```text
STOP NEW SIGNALS
```

If market data becomes stale:

```text
BLOCK EXECUTION
```

If order API fails:

```text
NO AUTOMATIC RETRY
```

If application restarts:

- Restore active signals
- Restore active positions
- Reconcile current state from Groww
- Do not place anything automatically

---

# 46. Development Phases

## Phase 1 — Groww connectivity

Build:

- Authentication
- Instrument master
- WebSocket
- REST market data
- Option chain
- Historical data
- Connection health

Deliverable:

```text
Reliable local Groww data collector
```

---

## Phase 2 — Local database

Build:

- Tick storage
- Candle aggregation/storage
- Option snapshots
- OI history
- IV/Greeks history

Deliverable:

```text
Local market-data archive
```

---

## Phase 3 — Indicator engine

Implement:

- VWAP
- EMA
- RSI
- ATR
- Volume statistics
- Swing detection
- Support/resistance

Deliverable:

```text
Market analysis engine
```

---

## Phase 4 — Market regime

Implement:

- Bullish
- Bearish
- Neutral
- Choppy
- Strong variants

Deliverable:

```text
15m + 5m regime engine
```

---

## Phase 5 — Strategy engine

Implement in order:

1. Breakout
2. Trend pullback
3. Breakdown
4. Reversal

Each strategy should have independent configuration and test results.

---

## Phase 6 — Option selector

Implement:

- ATM calculation
- Strike universe
- Delta filtering
- Liquidity filtering
- IV filtering
- OI analysis
- Contract scoring

Deliverable:

```text
Underlying setup → selected option contract
```

---

## Phase 7 — Backtester

Build:

- Historical replay
- Signal simulation
- Entry simulation
- Stop/target simulation
- Slippage
- Performance reports
- Walk-forward testing

Do not connect live execution yet.

---

## Phase 8 — Live scanner

Run strategies against live Groww data.

Output only:

```text
NO SIGNAL
WATCH
QUALIFIED
OPPORTUNITY
INVALIDATED
```

No orders.

---

## Phase 9 — Paper trading

Run the live scanner with simulated positions.

Measure real-time performance.

---

## Phase 10 — Dashboard

Build:

- Live chart
- Market regime
- Option chain
- Signals
- Trade proposal
- Risk calculator
- Paper positions
- Journal
- Statistics

---

## Phase 11 — Manual approval execution

Implement:

```text
Opportunity
    ↓
User approval
    ↓
Final validation
    ↓
Groww order
```

No automatic trading.

---

## Phase 12 — Post-trade analytics

Build:

- Trade journal
- Strategy statistics
- Signal statistics
- Time-of-day analysis
- Setup analysis
- Drawdown analysis
- User rejection analysis

---

# 47. Important Development Principles

## Principle 1 — No future data

The strategy must never access information that was not available at decision time.

## Principle 2 — Same engine everywhere

The same strategy code should power:

- Backtesting
- Historical replay
- Paper trading
- Live scanning

## Principle 3 — AI cannot override risk controls

AI is an explanation/analysis layer.

## Principle 4 — User approval is mandatory

No order can be submitted without an explicit approval action.

## Principle 5 — Final validation is mandatory

An approved but stale trade must not be submitted.

## Principle 6 — Every decision is logged

We need to know exactly why the system produced each signal.

## Principle 7 — Parameters must be testable

Do not hard-code thresholds throughout the application.

## Principle 8 — Start small

NIFTY + BANKNIFTY only.

Four strategies maximum for V1.

---

# 48. Definition of Done for V1

V1 is complete when the system can:

1. Connect to Groww.
2. Stream NIFTY/BANKNIFTY data.
3. Retrieve relevant option-chain information.
4. Store live data locally.
5. Calculate indicators.
6. Determine market regime.
7. Detect breakout/pullback/breakdown/reversal setups.
8. Select a suitable option contract.
9. Calculate entry, stop, target and R:R.
10. Calculate suggested quantity.
11. Generate an evidence-based opportunity.
12. Show it on the local dashboard.
13. Record the signal.
14. Paper-trade the signal.
15. Backtest the exact same strategy engine.
16. Allow user to reject or modify a proposal.
17. Require explicit user approval.
18. Revalidate the trade immediately before execution.
19. Place the approved Groww order.
20. Record the Groww order ID and execution result.
21. Monitor the resulting position.
22. Maintain a complete trade journal.

---

# 49. Initial Configuration

Start with:

```yaml
market:
  instruments:
    - NIFTY
    - BANKNIFTY

timeframes:
  regime: 15m
  setup: 5m
  confirmation: 1m

strategies:
  breakout: true
  pullback: true
  breakdown: true
  reversal: true

options:
  atm_strikes_each_side: 2
  preferred_delta_min: 0.40
  preferred_delta_max: 0.65

risk:
  minimum_rr: 1.5
  preferred_rr: 2.0
  max_positions: 1
  require_user_approval: true

session:
  start: "09:25"
  end: "15:15"

execution:
  enabled: false
  require_manual_approval: true
  final_validation: true
  automatic_retry: false
```

Execution should remain disabled during development, backtesting and paper trading.

---

# 50. Final Product Philosophy

The application should not behave like:

> "AI says BUY."

It should behave like:

> **"A statistically testable setup has appeared. Here is the exact contract, evidence, entry zone, invalidation, target, risk, quantity and reasons. Review it and decide."**

The final decision remains with the user.

The system's job is to make the opportunity **faster to identify, easier to evaluate, measurable, reproducible and safer to execute**.


---

# 51. Extended Risk & Portfolio Controls

The following controls are mandatory additions to the V1 architecture.

## 51.1 Portfolio-Level Greeks

Track portfolio exposure across all active positions, not only individual trades.

At minimum:

- Net Delta
- Net Gamma
- Net Theta
- Net Vega

Example:

```text
NIFTY CE position
        +
BANKNIFTY CE position
        ↓
Portfolio exposure
        ↓
Net Delta / Gamma / Theta / Vega
```

The risk engine must evaluate the combined portfolio before approving a new trade.

A trade that is acceptable individually may be rejected or downgraded if it creates excessive combined exposure.

The thresholds must be configurable.

---

## 51.2 Adaptive Position Sizing / Losing-Streak Cooldown

Implement a separate cooldown mechanism from the hard daily-loss limit.

Example:

```text
Normal size
    ↓
Loss #1
    ↓
Loss #2
    ↓
Loss #3
    ↓
Reduced size / cooldown
```

Configurable parameters:

```yaml
cooldown:
  enabled: true
  consecutive_losses_trigger: 3
  size_multiplier: 0.50
  recovery_wins_required: 1
```

Possible states:

```text
NORMAL
REDUCED_SIZE
COOLDOWN
```

The system must never increase risk to recover previous losses.

The exact parameters must be validated through backtesting and paper trading.

---

## 51.3 Correlation Guard

NIFTY and BANKNIFTY can create overlapping directional exposure.

Before approving a new position:

```text
Existing NIFTY exposure
        +
Proposed BANKNIFTY exposure
        ↓
Correlation / directional exposure check
```

The system should consider:

- Existing position direction
- Net portfolio delta
- Underlying correlation
- Proposed option delta
- Portfolio-level risk

Example:

```text
NIFTY CALL already open
+
BANKNIFTY CALL proposal
+
High combined directional exposure
=
BLOCK / DOWNGRADE
```

Do not rely solely on a static correlation coefficient. Evaluate current portfolio exposure.

---

## 51.4 Time-Based Signal Decay

A signal must expire even if price remains inside the original entry zone.

Each proposal receives:

- Created timestamp
- Expiry timestamp
- Price invalidation
- Time invalidation

Example:

```text
Signal generated: 10:32
Signal valid until: 10:42

At 10:42:
SIGNAL EXPIRED
```

Configurable:

```yaml
signal_decay:
  default_minutes: 10
```

Different strategy types may use different decay periods.

A stale signal must never remain permanently actionable.

---

# 52. Strategy Validation & Statistical Monitoring

## 52.1 Score Calibration

The signal score must be validated against actual outcomes.

Store every completed signal with:

- Score
- Strategy
- Direction
- Market regime
- Outcome
- R result
- MFE
- MAE

Create score buckets:

```text
60–69
70–79
80–89
90–100
```

Periodically calculate:

- Bucketed win rate
- Average R
- Expectancy
- Profit factor
- Calibration error
- Brier score where probability estimates are available

Example:

```text
Score bucket    Signals    Win rate    Avg R
60–69           ...        ...         ...
70–79           ...        ...         ...
80–89           ...        ...         ...
90–100          ...        ...         ...
```

The purpose is to determine whether higher scores actually correspond to better historical outcomes.

Do not describe the score as a probability unless it has been statistically calibrated.

---

## 52.2 Live vs Backtest Drift Detector

Compare live paper-trading performance against the expected backtest/validation distribution.

Monitor:

- Win rate
- Expectancy
- Average R
- Profit factor
- Drawdown
- Signal frequency
- Slippage
- MFE/MAE
- Strategy-specific performance

Example:

```text
Backtest expectancy: +0.32R
Live paper expectancy: -0.08R

→ DRIFT WARNING
```

Possible states:

```text
NORMAL
WATCH
DRIFT WARNING
CRITICAL DRIFT
```

Initial implementation should flag the strategy rather than automatically change parameters.

A configurable policy may move a critically drifting strategy to observation-only mode.

---

## 52.3 Shadow Testing

Allow a new strategy configuration to run silently alongside the active configuration.

Example:

```text
LIVE:
Breakout v1.0
Volume threshold = 1.30x

SHADOW:
Breakout v1.1
Volume threshold = 1.50x
```

Both receive identical live data.

Only the active configuration can generate actionable proposals.

Shadow configuration records:

- Signals
- Hypothetical entries
- Hypothetical exits
- R results
- Missed opportunities
- Performance statistics

This allows strategy changes to be evaluated without deploying them blindly.

---

# 53. Event Awareness Layer

Create a dedicated Market Event Engine.

## Event categories

Monitor relevant events including:

- RBI policy decisions
- Union Budget
- US Federal Reserve decisions
- US CPI
- US employment data
- Major Indian macro releases
- Important global macro events
- Relevant exchange holidays
- NIFTY expiry
- BANKNIFTY expiry
- Other F&O expiries where relevant

The event engine should provide:

```text
NORMAL
CAUTION
EVENT RISK
BLOCK NEW SIGNALS
```

Event policies must be configurable by event type.

Example:

```yaml
events:
  fed_decision:
    before_minutes: 30
    after_minutes: 30
    action: block_new_signals

  expiry:
    action: expiry_specific_rules
```

Do not assume every event should block trading. Validate event-specific behaviour historically.

The event layer should primarily prevent the strategy from interpreting event-driven volatility as ordinary technical behaviour.

---

# 54. Execution & Market-Data Robustness

## 54.1 WebSocket Reconnection

Implement robust connection recovery.

Required:

- Connection health monitoring
- Exponential backoff
- Jitter
- Authentication refresh when necessary
- Automatic resubscription
- Subscription-state tracking
- Stale-data detection

Example:

```text
Disconnect
   ↓
1 sec
   ↓
2 sec
   ↓
4 sec
   ↓
8 sec
   ↓
16 sec
   ↓
Maximum retry interval
```

Do not generate signals while the data feed is unhealthy.

---

## 54.2 Tick/Data Sanity Filter

Raw market data must pass validation before reaching the indicator engine.

Reject or flag:

- Zero prices
- Negative prices
- Impossible price jumps
- Duplicate ticks
- Stale timestamps
- Future timestamps
- Out-of-order timestamps
- Invalid bid/ask relationships
- Impossible OI changes
- Invalid volume values
- Disconnected/stale feeds

Pipeline:

```text
Groww WebSocket
      ↓
Raw Tick
      ↓
Data Sanity Filter
      ↓
Validated Tick
      ↓
Indicator Engine
```

Bad data must not create a trading signal.

---

## 54.3 Protective Server-Side Stop

Investigate Groww's currently supported F&O order types before implementation.

If Groww supports an appropriate broker-side protective mechanism such as GTT/bracket/linked stop functionality for the exact intended F&O workflow, the preferred architecture is:

```text
User approves
      ↓
Entry order
      ↓
Fill confirmed
      ↓
Protective stop placed server-side
      ↓
Target remains manually controlled
```

This is intended as a safety net if the local application disconnects after entry.

Do not claim a local stop is broker-side protection.

Do not implement this until the exact Groww API/order semantics have been verified.

---

# 55. Operations & Auditability

## 55.1 Configuration Change Log

Every strategy/risk configuration change must be recorded.

Record:

- Timestamp
- User
- Configuration version
- Parameter
- Old value
- New value
- Reason/comment

Example:

```text
24 Sep 2026 11:42

Parameter:
breakout_volume_multiplier

Old:
1.30

New:
1.40

Reason:
Walk-forward validation

Config:
v1.3.2
```

All backtests, signals and trades must reference the configuration version used.

---

## 55.2 End-of-Day Report

Automatically generate a daily summary.

Deliver through the dashboard and optionally Telegram.

Example:

```text
DAILY TRADING REPORT
24 Sep 2026

Signals detected: 17
Qualified: 6
Rejected: 8
Invalidated: 3

Trades:
2 approved
2 executed
1 winner
1 loser

P&L: ₹XXX
Max drawdown: ₹XXX

Strategy performance:
Breakout: ...
Pullback: ...
Breakdown: ...
Reversal: ...

Market:
NIFTY: ...
BANKNIFTY: ...

Data quality:
99.8%

Strategy health:
NORMAL
```

The report must distinguish:

- Signals generated
- Signals rejected by user
- Signals invalidated
- Trades approved
- Orders executed
- Trades closed

---

# 56. Broker & Data Provider Abstraction

Even though Groww is the V1 provider, strategy code must not directly depend on Groww-specific API calls.

Create separate interfaces.

## MarketDataProvider

```python
class MarketDataProvider:
    def get_quote(...)
    def get_option_chain(...)
    def get_historical_candles(...)
    def subscribe(...)
    def unsubscribe(...)
```

## BrokerAdapter

```python
class BrokerAdapter:
    def get_positions(...)
    def get_orders(...)
    def place_order(...)
    def modify_order(...)
    def cancel_order(...)
    def get_order_status(...)
    def place_protective_stop(...)
```

Groww implementation:

```text
GrowwMarketDataProvider
GrowwBrokerAdapter
```

Future implementations can include:

```text
OtherMarketDataProvider
OtherBrokerAdapter
```

This allows changing or adding data vendors/brokers without changing strategy logic.

---

# 57. Expanded System Architecture

```text
                         ┌─────────────────┐
                         │      GROWW      │
                         └────────┬────────┘
                                  │
                      Market Data Provider
                                  │
                                  ↓
                 ┌────────────────────────────┐
                 │ DATA HEALTH / QUALITY      │
                 │                            │
                 │ Reconnect                  │
                 │ Resubscribe                │
                 │ Tick validation            │
                 │ Stale-data detection       │
                 └─────────────┬──────────────┘
                               ↓
                 ┌────────────────────────────┐
                 │    MARKET DATA ENGINE      │
                 └─────────────┬──────────────┘
                               ↓
              ┌────────────────┴────────────────┐
              ↓                                 ↓
      Market Regime Engine               Event Engine
              ↓                                 ↓
              └────────────────┬────────────────┘
                               ↓
                 ┌────────────────────────────┐
                 │      STRATEGY ENGINE       │
                 │                            │
                 │ Breakout                   │
                 │ Pullback                   │
                 │ Breakdown                  │
                 │ Reversal                   │
                 └─────────────┬──────────────┘
                               ↓
                 ┌────────────────────────────┐
                 │     OPTION SELECTOR        │
                 │                            │
                 │ OI / IV / Greeks           │
                 │ Delta / Liquidity          │
                 │ Volume / Contract score    │
                 └─────────────┬──────────────┘
                               ↓
                 ┌────────────────────────────┐
                 │       RISK ENGINE          │
                 │                            │
                 │ Position sizing            │
                 │ Portfolio Greeks           │
                 │ Correlation guard          │
                 │ Losing-streak cooldown     │
                 │ Daily loss limit           │
                 └─────────────┬──────────────┘
                               ↓
                 ┌────────────────────────────┐
                 │      TRADE PROPOSAL        │
                 │                            │
                 │ Entry / SL / Target        │
                 │ Qty / Risk / R:R           │
                 │ Evidence / Expiry          │
                 └─────────────┬──────────────┘
                               ↓
                              USER
                       ┌────────┼────────┐
                       ↓        ↓        ↓
                    REJECT    MODIFY   APPROVE
                                         ↓
                              FINAL VALIDATION
                                         ↓
                                Broker Adapter
                                         ↓
                                       GROWW
                                         ↓
                              Protective Stop*
                                         ↓
                                Position Monitor
```

`*` Only if the relevant Groww broker-side order functionality is verified and suitable.

---

# 58. Research & Validation Architecture

Run a parallel research system:

```text
Historical Data
      ↓
Backtester
      ↓
Validation
      ↓
Calibration
      ↓
Drift Detection
```

And during live operation:

```text
Live Data
   ├── Active Configuration
   │       ↓
   │   Live/Paper Signals
   │
   └── Shadow Configuration
           ↓
       Silent Signals
```

This allows strategy improvements without risking live capital.

---

# 59. Additional Database Tables

Add:

## portfolio_risk_snapshots

- timestamp
- net_delta
- net_gamma
- net_theta
- net_vega
- gross_exposure
- directional_exposure

## strategy_scores

- signal_id
- score
- score_bucket
- outcome
- r_multiple
- calibration_version

## strategy_versions

- version
- strategy
- parameters_json
- created_at
- created_by
- reason

## config_change_log

- timestamp
- user
- config_version
- parameter
- old_value
- new_value
- reason

## events

- event_id
- event_type
- event_time
- severity
- pre_window
- post_window
- action

## shadow_signals

- timestamp
- strategy
- shadow_version
- hypothetical_contract
- hypothetical_entry
- hypothetical_stop
- hypothetical_target
- outcome

## data_health

- timestamp
- provider
- connection_status
- last_tick_time
- latency
- stale
- rejected_tick_count

## strategy_health

- timestamp
- strategy
- backtest_expectancy
- live_expectancy
- drift_status
- sample_size

---

# 60. Expanded Risk Configuration

Example:

```yaml
risk:
  minimum_rr: 1.5
  preferred_rr: 2.0

  max_positions: 1

  max_daily_loss: configurable
  max_trade_risk_percent: configurable

  portfolio_greeks:
    enabled: true
    max_net_delta: configurable
    max_net_gamma: configurable
    max_net_theta: configurable
    max_net_vega: configurable

  correlation_guard:
    enabled: true
    max_combined_directional_exposure: configurable

  cooldown:
    enabled: true
    consecutive_losses_trigger: 3
    size_multiplier: 0.50
    recovery_wins_required: 1
```

Never hard-code monetary risk values into strategy logic.

---

# 61. Expanded Signal Configuration

Example:

```yaml
signal:
  minimum_score: 70

  decay:
    enabled: true
    default_minutes: 10

  liquidity:
    max_spread_percent: 1.5

  option_selection:
    atm_strikes_each_side: 2
    preferred_delta_min: 0.40
    preferred_delta_max: 0.65

  breakout:
    volume_multiplier: 1.30

  risk_reward:
    minimum: 1.5
    preferred: 2.0
```

Every signal must store the complete effective configuration snapshot/version.

---

# 62. Expanded System States

The application should expose clear global states:

```text
STARTING
CONNECTING
LIVE
DATA_DEGRADED
DATA_STALE
MARKET_CLOSED
EVENT_CAUTION
TRADING_BLOCKED
COOLDOWN
PAPER_MODE
LIVE_APPROVAL_MODE
```

The user should always be able to see the current state.

---

# 63. Execution State Machine

Use an explicit state machine.

```text
SIGNAL_CREATED
      ↓
QUALIFIED
      ↓
PROPOSAL_ACTIVE
      ↓
USER_APPROVED
      ↓
FINAL_VALIDATION
      ↓
ORDER_SUBMITTED
      ↓
ORDER_FILLED
      ↓
POSITION_ACTIVE
      ↓
EXIT_PROPOSAL
      ↓
USER_APPROVED_EXIT
      ↓
EXIT_ORDER
      ↓
TRADE_CLOSED
```

Failure states:

```text
INVALIDATED
REJECTED
VALIDATION_FAILED
ORDER_FAILED
EXPIRED
CANCELLED
```

Never infer execution state from UI state.

Always reconcile with Groww order/position data.

---

# 64. Final Expanded V1 Definition of Done

V1 is complete when the system can:

1. Connect to Groww.
2. Maintain a resilient WebSocket connection.
3. Automatically reconnect and resubscribe.
4. Validate incoming market data.
5. Detect stale or abnormal data.
6. Retrieve NIFTY/BANKNIFTY live data.
7. Retrieve relevant option-chain information.
8. Store live market data locally.
9. Store historical data required for research.
10. Calculate indicators.
11. Determine market regime.
12. Detect breakout setups.
13. Detect trend-pullback setups.
14. Detect breakdown setups.
15. Detect reversal setups.
16. Select suitable option contracts.
17. Analyze OI.
18. Analyze IV.
19. Analyze Greeks.
20. Analyze liquidity.
21. Calculate signal score.
22. Apply hard rejection filters.
23. Apply time-based signal decay.
24. Calculate entry/stop/target.
25. Calculate R:R.
26. Calculate position size.
27. Apply losing-streak cooldown.
28. Apply portfolio-level Greek limits.
29. Apply correlation guard.
30. Apply event-awareness rules.
31. Generate trade proposals.
32. Allow user rejection.
33. Allow quantity modification.
34. Require explicit approval.
35. Revalidate the proposal immediately before execution.
36. Place the approved order through the Groww broker adapter.
37. Optionally place a verified broker-side protective stop after fill.
38. Monitor positions.
39. Record every signal.
40. Record every rejection.
41. Record every approval.
42. Record every order and execution.
43. Maintain a complete trade journal.
44. Run historical backtests.
45. Run historical replay.
46. Run live paper trading.
47. Calculate score calibration.
48. Detect live-vs-backtest drift.
49. Run shadow strategy configurations.
50. Maintain configuration version history.
51. Maintain a complete configuration audit log.
52. Generate end-of-day reports.
53. Send configurable Telegram alerts/reports.
54. Keep strategy logic independent from Groww.
55. Keep market-data and broker interfaces replaceable.
56. Fail safely when data or broker connectivity is unhealthy.
57. Never place an order without explicit user approval.

---

# 65. Final Product Philosophy

The system must not behave like:

> "AI says BUY."

It should behave like:

> **"A statistically testable setup has appeared. Here is the exact contract, evidence, entry zone, invalidation, target, risk, portfolio exposure, current event context and quantity. Review it and decide."**

The system should make opportunities:

- Faster to identify
- Easier to evaluate
- Statistically measurable
- Reproducible
- Auditable
- Safer to execute

The user remains the final decision-maker for every live trade.
