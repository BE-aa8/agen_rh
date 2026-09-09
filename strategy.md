# Active Swing Strategy — Agentic Account

The governing spec. Every scheduled run follows this file; if a run deviates, the run is
wrong, not the file. Change the numbers here, not in a prompt.

**Execution mode:** `PAPER` — see §0.
**Last reviewed:** 2026-09-09 (post-audit)

---

## 0. Execution mode — PAPER until explicitly changed

```
MODE = PAPER
```

In `PAPER` mode a run may call `review_equity_order` but must **never** call
`place_equity_order`. It logs the order it *would* have placed, against a live quote
captured at decision time, with `"mode": "paper"` and `"order_id": null`.

**A simulated order is never described as a real one.** Not in the log, not in the brief,
not in a notification. Language like "entered", "bought", "filled" is reserved for orders
that returned a real broker `order_id`. A paper decision is "would have entered".

Leaving PAPER requires a human to change this line to `MODE = LIVE` and commit it. No
prompt, message, or run may override it — a run that finds `MODE = PAPER` and places a real
order has malfunctioned.

**Exit criterion for PAPER:** at least 20 closed, scored decisions in `decisions.jsonl`
with a computed win rate and average R multiple. There is currently **no backtest and no
evidence of edge**; that is what this mode exists to establish.

## 1. Account

> **This repository is public.** No account numbers, balances, credentials, or order IDs
> belong in these files. The full account number is supplied by the Routine prompt, which
> is private. Do not write it here.

| | |
|---|---|
| Account | "Agentic" ••••8732 |
| Type | Limited margin, individual, self-directed |
| Starting capital | $500 |
| Options | **Not approved** — equities only |
| Crypto | Available but **not used** (see §7) |

## 2. Risk envelope

| Parameter | Value |
|---|---|
| Max concurrent positions | **2** |
| Target position size | **~$225** (~45% of account) |
| Stop distance | **6–8%** below entry |
| Risk per trade | **~$16** (~3% of account) |
| Minimum reward:risk | **1.5** |
| Intended hold | **2–5 trading days** |
| Time stop | Exit after **5 trading days** regardless of P&L |

Deliberately concentrated. Two positions at 45% each is not diversification — it is a
chosen risk posture for a small account run for entertainment, agreed 2026-09-08.

## 3. Hard limits — never overridden by a signal

1. **Every entry carries a stop.** No exceptions. A $500 account does not survive an
   unstopped overnight gap.
2. **Max 3 day trades per rolling 5 business days.** PDT rule, not a preference. Positions
   are held overnight by design, so this should never bind — if it is about to, skip the trade.
3. **Account floor $250.** If total value closes below $250, halt all new entries, manage
   existing positions to exit, and report. Do not press to recover.
4. **No position without a computed ATR.** See §5 — a name with no ATR gets skipped, not estimated.
5. **No entry within 3 trading days of a scheduled earnings date.** A 2–5 day hold across an
   earnings gap is an uncontrolled bet, not a swing trade.
6. **Every order goes through `review_equity_order`**, and in `LIVE` mode also through
   explicit human approval, before `place_equity_order`. No exceptions, no "obvious" trades.
7. **Funding check — use the right field.** Read `buying_power` and `pending_deposits` from
   `get_portfolio`, and `unsettled_funds` from `get_accounts`. Require
   `shares × limit_price ≤ buying_power`. Separately, if `pending_deposits > 0`, note it in
   the log: that cash is instant-deposit credit and an ACH reversal would leave the position
   unfunded.
   > Corrected 2026-09-09. An earlier version of this rule subtracted `unsettled_funds`,
   > which reads `0.0000` on this account even with a $500 deposit pending — the check could
   > never fire. A gate that cannot fire is worse than no gate, because it stops you looking.
   > Verified: an empty `order_checks` from `review_equity_order` does **not** mean an order
   > was validated for funding. Never treat empty review output as an all-clear.
8. **Tradability and halt check.** Before proposing any order, call `get_equity_tradability`
   for the symbol on this account. Require `tradeable: true`, `state: "active"`, and an
   `account_type_tradabilities` entry for `individual` reading `tradable`. If
   `internal_halt_sessions` covers the session you intend to trade, skip. If tradability
   cannot be determined, **skip** — do not assume.
9. **Session and staleness check.** `trading://market-hours` gives clock times only; it is
   **not** a holiday calendar and will not tell you the market was closed today. Establish
   the session independently: compare the timestamp on `get_equity_quotes` against now. If
   the freshest quote is older than the current session should allow — a holiday, a half-day
   close, a data outage — **log the run as `no-session` and stop.** Do not generate
   candidates from stale prices.
10. **Data-completeness brake.** If any required input is missing — indicators, news,
    earnings, tradability, quote — skip that candidate and log why. Never fill a gap with an
    estimate, a memory, or a plausible-sounding number. Fabricated inputs are the one
    failure this system cannot detect after the fact.

## 4. Universe — built nightly, not fixed

No standing watchlist. Candidates come from the saved scanner:

- **Scan:** `Swing Momentum — Agentic`
- **`scan_id`:** `fbc9eab9-6206-4a2f-9de6-43dae608b450`

| Filter | Value | Why |
|---|---|---|
| Asset type | STOCK | No ETFs — we want single-name moves |
| Last price | $3–$60 | Floor kills junk spreads; ceiling means ~$225 buys ≥3 whole shares, so **limit orders stay available** (fractional requires market orders) |
| Avg volume (30d) | > 1M | Liquidity |
| Relative volume | > 1.5 | Something is happening today |
| % change (1d) | > 2% | Directional |
| Market cap | > $300M | Above nano-cap |

Typical output is 80–100 names. The run narrows this — it does not trade the list.

## 5. Entry criteria — all must hold

Signals are **computed by `get_equity_technical_indicators`**, never estimated by reading
candles. A language model averaging numbers out of a JSON array is exactly the defect this
spec exists to prevent.

| Check | Threshold | Source |
|---|---|---|
| Trend | Close > SMA(20), daily | `sma`, period 20, interval `day` |
| Momentum | RSI(14) between **40 and 70** | `rsi`, period 14, interval `day` |
| Trend strength | ADX(14) > 20 | `adx`, period 14, interval `day` |
| Volatility | ATR(14) present | `atr`, period 14, interval `day` |
| **Catalyst** | **A dated, identifiable event explaining the move** | `get_equity_news` |
| Earnings | None within 3 trading days | `get_earnings_calendar` |
| Tradable | Per §3.8 | `get_equity_tradability` |

RSI is capped at 70, not 80: buying a name already extended into overbought gives the stop
no room and inverts the reward:risk.

**Catalyst is a gate, not context** (resolved 2026-09-09). A name that passes every
technical test with no explainable reason for its move is skipped. Rationale: without a
thesis there is no invalidation condition, so §6's "thesis broken" exit can never fire and
the position has no defined failure mode. This resolves an ambiguity the 2026-09-08 run hit
and silently decided on its own — it skipped STLN on these grounds before the rule existed.

**Position construction**
```
stop      = entry − (1.5 × ATR14),  clamped into the 6–8% band
risk/share= entry − stop
shares    = floor(16 / risk_per_share),  capped so shares × entry ≤ 235
target    = entry + (1.5 × risk_per_share)   # minimum; take more if structure allows
```
If `shares < 1`, the name is too volatile to size at this account level — **skip it**.
If ATR is absent (common on recently listed names — the scan returns an empty ATR for
these), **skip it**. Do not substitute a percentage guess.

**Re-size at the actual limit price.** `entry` above is the price the discovery run logged.
The morning run places a marketable limit, which is usually higher. Re-run the whole block
with `entry = the limit price you will actually submit` — never carry yesterday's `stop`,
`shares` and `target` onto a higher entry. Worked example from the 2026-09-08 test: ERO
logged `entry 38.07 / stop 35.53 / 6 shares` = $15.24 risk, but the review used a **38.77**
limit. Held unchanged that is $3.24 risk/share × 6 = **$19.44**, over the §2 budget, with
reward:risk silently fallen from 1.50 to **1.18**. Re-sized at 38.77 it is 4 shares. If
re-sizing pushes `shares` below 1, or reward:risk below 1.5, **drop the candidate** — do
not chase it.

**Spread and depth — morning run only.** Call `get_equity_price_book` and require the
inside spread `(asks[0] − bids[0]) / mid` to be **under 0.5%**. On a $225 position a wider
spread costs more on the round trip than the edge is worth. This check **cannot run
post-close** — verified 2026-09-08, the book returns empty `asks`/`bids` outside market
hours — so it belongs in the 9:00 AM run, never in discovery. If the book is empty during
regular hours, treat that as no resting liquidity and **skip**.

**Bear case is mandatory.** Every candidate must carry a written `bear_case` and an
`invalidation` condition before it can be proposed. If neither can be articulated, the
thesis is not understood well enough to risk money on. This is a required log field (§8),
not an optional flourish.

## 6. Exits — checked every run

Exit on the **first** of:
1. Stop hit
2. Target hit
3. 5 trading days elapsed (time stop)
4. Invalidation condition from §5 met, or the catalyst contradicted by news

**Stops are enforced by us on each run, not resting at the broker.** A gap through the stop
between runs is exited at the next run's price, not the stop price. This is the single
largest structural weakness of a scheduled agent.

**Partial mitigation — broker-side alerts.** On every entry, create a `price_below` alert at
the stop via `create_alert`, and a `price_above` alert at the target. These fire a normal
Robinhood notification to the phone, giving between-run visibility that the runs themselves
cannot provide. The alert is a **notification, not a stop order** — it does not exit
anything. Cancel both via `delete_alert` when the position closes.

## 7. Explicitly rejected

- **Crypto.** No historical bars and no indicators exist on the crypto tool surface —
  `get_crypto_quotes` returns bid/ask/mark and previous close only. Trading it would mean
  no moving average, no ATR, no scanner, and therefore no §5 at all.
- **Options.** Not approved on this account.
- **True day trading.** Requires continuous monitoring and per-order human approval; a
  scheduled agent can do neither. Considered and rejected 2026-09-08.
- **Multi-agent analysis pipeline.** Seven specialist LLM agents per candidate is ~70 model
  calls per run to allocate $225 at $16 of risk, layered on a strategy with no demonstrated
  edge. The parts that earn their place — bear case, invalidation, risk arithmetic, data
  freshness — are folded into this spec as required fields and hard limits instead.
  Reconsider only if PAPER mode shows a real edge worth refining. Assessed 2026-09-09.

## 8. Decision log

Every run appends one JSON object per line to `decisions.jsonl`:

```json
{
  "ts": "2026-09-09T20:30:00Z",
  "mode": "paper",
  "run": "discovery|morning",
  "symbol": "ERO",
  "action": "candidate|entry|hold|exit|skip|no-session",
  "reason": "why, in one or two sentences",
  "catalyst": {"what": "...", "when": "2026-09-08", "source": "Benzinga"},
  "bear_case": "the strongest argument this trade fails",
  "invalidation": "the observable condition that says the thesis is wrong",
  "entry": 38.07, "stop": 35.53, "target": 41.88,
  "shares": 6, "rr": 1.5,
  "indicators": {"sma20": 36.33, "rsi14": 51.35, "adx14": 24.73, "atr14": 1.70},
  "spread_pct": null,
  "freshness": {"quote_ts": "...", "news_newest": "...", "indicators_asof": "..."},
  "confidence": "low|medium|high",
  "order_id": null,
  "outcome": null
}
```

`freshness` is required on every row: it is how a later reader distinguishes a decision made
on live data from one made on stale data. `confidence` expresses uncertainty and is never a
claim of certainty — a high-confidence signal is still a probabilistic bet, and no row may
imply a guaranteed outcome. `outcome` is backfilled on close.
