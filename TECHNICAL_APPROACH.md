# Technical Approach: Automated Prediction-Market Arbitrage

## 1) Goals, Scope, and Success Criteria

### Goals
- Continuously scan multiple prediction market venues.
- Detect cross-venue mispricings for **equivalent outcomes**.
- Execute offsetting trades to capture a **net positive spread after costs**.
- Maintain near-zero net exposure while tracking PnL, inventory, and system health.

### Scope (MVP)
- 2–3 venues with stable APIs.
- A single asset class (binary markets with “Yes/No”).
- Real-time or near-real-time order book ingestion.
- Automated execution with **risk controls and safeguards**.

### Success Criteria
- Consistent detection of arbitrage opportunities with **positive post-fee spread**.
- Successful placement of offsetting trades without leaving significant residual exposure.
- Auditable logs and monitoring for decisions, trades, and failures.

## 2) Architecture Overview

```
+----------------------+     +------------------+     +------------------+
| Venue Connectors     | --> | Market Normalizer| --> | Market Matcher   |
| (API + Auth)         |     | & Canonicalizer  |     | (equivalence map)|
+----------------------+     +------------------+     +------------------+
            |                          |                        |
            v                          v                        v
+----------------------+     +------------------+     +------------------+
| Pricing Engine       | --> | Risk Engine      | --> | Execution Engine |
| (spreads + costs)    |     | (liquidity, TTL) |     | (orders)         |
+----------------------+     +------------------+     +------------------+
            |                          |                        |
            v                          v                        v
+--------------------------------------------------------------------------+
| Storage & Monitoring (PnL, orders, inventory, alerts, metrics, logs)     |
+--------------------------------------------------------------------------+
```

### Primary Data Flows
1. **Venue Connectors** ingest markets and order books.
2. **Normalizer** standardizes metadata into canonical schemas.
3. **Matcher** links equivalent markets across venues with confidence scores.
4. **Pricing Engine** computes actionable spreads after costs.
5. **Risk Engine** validates liquidity, TTL, and compliance thresholds.
6. **Execution Engine** places time-bounded, offsetting trades and reconciles fills.
7. **Monitoring & Storage** tracks state, PnL, and alerts.

## 3) Domain Model & Data Contracts

### Canonical Entities
- **Venue**: name, API base URL, auth method, rate limits, fee schedules.
- **Market**: venue_market_id, event_name, outcome_set, resolution_date, rules_text.
- **Outcome**: canonical_outcome_id (e.g., YES/NO), display_name.
- **OrderBookSnapshot**: bid_levels, ask_levels, timestamp, venue.
- **MatchedMarket**: canonical_market_id, mapped_venue_markets[], confidence_score.
- **ArbOpportunity**: matched_market_id, side, venue_buy, venue_sell, spread_gross, spread_net.
- **TradeIntent**: opportunity_id, size, limit_prices, TTL, risk_checks.
- **Execution**: orders placed, fills, fees, resulting inventory.

### Required Metadata Fields
- `event_title` (normalized string)
- `resolution_date` (UTC)
- `market_type` (binary)
- `rules_text` or equivalent terms
- `contract_spec` (e.g., “YES pays $1 if outcome occurs”)

### Data Storage (MVP)
- **Relational DB** for market mappings, orders, fills, and PnL.
- **Time-series store** or table for snapshots and spreads.
- **Audit log** for all decisions and API responses.

## 4) Market Discovery & Normalization

### Venue Connector Responsibilities
- Authenticate (API key / OAuth / session token).
- Pull active markets.
- Pull best bid/ask (or depth) for each market.
- Handle rate limits, retries with exponential backoff.

### Normalization Rules
- **Event name**: lowercased, stripped punctuation, normalized dates.
- **Outcome labels**: map to canonical YES/NO.
- **Resolution time**: parse and convert to UTC.
- **Rules text**: hash to detect near-identical text across venues.

### Outputs
- `NormalizedMarket` objects for matching.
- `NormalizedOrderBook` snapshots for pricing.

## 5) Market Matching (Equivalence Mapping)

### Matching Logic (Prescriptive)
1. **Hard filters**
   - Same market type (binary).
   - Resolution date within a tolerance (e.g., ±12h).
   - Event title similarity above threshold.
2. **Confidence scoring**
   - Title similarity (e.g., cosine similarity with tokenization).
   - Rules text similarity (hash or fuzzy match).
   - Outcome label alignment.
3. **Manual overrides**
   - Store `matched_market_id` overrides for ambiguous pairs.
   - Allow operators to lock mappings.

### Output
- `MatchedMarket` with confidence score and mapping provenance.

## 6) Pricing & Spread Calculation

### Pricing Inputs
- Best bid/ask per venue for each matched market.
- Fee schedules (maker/taker) and minimum order sizes.
- Slippage model (static or based on depth).

### Spread Calculation (Prescriptive)
For each matched market and direction:
- **Buy YES on Venue A, Sell YES on Venue B**:
  - Gross spread = (best_bid_B - best_ask_A).
  - Net spread = gross spread - fees - estimated slippage.
- Require `net_spread > min_profit_threshold`.

### TTL and Staleness
- Discard data older than configurable TTL (e.g., 2–5 seconds).
- If one side is stale, skip opportunity.

### Output
- `ArbOpportunity` objects with computed net spread.

## 7) Risk Engine

### Hard Risk Checks (Must Pass)
- **Liquidity**: minimum available size on both venues.
- **Max exposure**: per-venue and global limits.
- **Time to resolution**: ensure adequate settlement margin.
- **Market status**: not paused, not near close.
- **Data freshness**: TTL check for both order books.

### Soft Risk Checks (Advisory)
- **Volatility score** from recent spread variance.
- **API health**: degrade or pause on elevated error rate.

### Output
- `TradeIntent` with approved size, price limits, and TTL.

## 8) Execution Engine

### Execution Strategy (Prescriptive)
1. **Pre-flight**: lock opportunity and re-check order book.
2. **Place orders** in both venues (time-bounded).
3. **Handle partial fills**:
   - If one side fills and the other doesn’t, attempt to hedge or unwind quickly.
4. **Cancel & reprice** if TTL exceeded.
5. **Reconcile** fills, fees, and final net exposure.

### Atomic-Like Safeguards
- Use **short TTL** for both orders.
- Place orders in order of venue reliability (higher fill probability first).
- Consider using **IOC (Immediate-or-Cancel)** or **FOK (Fill-or-Kill)** where available.

### Order Sizing
- Size based on **min depth** across venues.
- Apply risk multiplier (e.g., 0.5–0.8) to reduce market impact.

## 9) Inventory & PnL Management

### Inventory Rules
- Track position by venue and outcome.
- Maintain **net exposure near zero**.
- Define auto-hedge actions if exposure exceeds threshold.

### PnL Calculation
- Include fees, slippage, funding, and settlement payouts.
- Attribute PnL by opportunity and by venue.

## 10) Monitoring, Alerts, and Reporting

### Metrics to Track
- Opportunity count, accepted trades, fill rates.
- Net exposure, per-venue exposure, PnL.
- API errors, latency, and stale data counts.

### Alerts
- Failure to place one side of a trade.
- Stale data above threshold.
- Exposure above limits.
- API rate limit or authentication failures.

## 11) Implementation Plan (Step-by-Step)

### Phase 1: Foundations
1. Define canonical schemas and DB tables.
2. Build a single venue connector with market + order book ingestion.
3. Implement normalization and storage pipeline.

### Phase 2: Matching & Pricing
1. Add second venue connector.
2. Implement matcher and confidence scoring.
3. Build pricing engine to emit opportunities.

### Phase 3: Risk & Execution
1. Implement risk engine with hard checks.
2. Add execution engine (order placement + cancellation).
3. Implement fill reconciliation and inventory tracking.

### Phase 4: Monitoring & Ops
1. Add metrics and alerting.
2. Build operator dashboard for mappings and overrides.
3. Run paper-trading / sandbox environment.

## 12) Non-Functional Requirements

- **Reliability**: recover from API errors, retries, and backoff.
- **Latency**: end-to-end opportunity evaluation within seconds.
- **Security**: secrets management and audit logging.
- **Compliance**: confirm venue ToS and regulatory requirements.

## 13) Open Questions & Risks

- Venue API stability and order book depth accuracy.
- Latency-sensitive race conditions between venues.
- Regulatory constraints in target jurisdictions.
- Slippage modeling accuracy for thin markets.

## 14) Deliverables

- Technical design doc (this document).
- Canonical schemas and DB migrations.
- Venue connectors and normalized data ingestion.
- Market matcher with confidence scoring and manual overrides.
- Pricing, risk, and execution engines.
- Monitoring and reporting tools.

