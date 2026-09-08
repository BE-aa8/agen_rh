# Active Swing Strategy — Agentic Account

The governing spec. Every scheduled run follows this file; if a run deviates, the run is
wrong, not the file. Change the numbers here, not in a prompt.

**Status:** not yet live — awaiting first scheduled run.
**Last reviewed:** 2026-09-08

---

## 1. Account

> **This repository is public.** No account numbers, balances, credentials, or order IDs
> belong in these files. The full account number is supplied by the Routine prompt, which
> is private. Do not write it here.

| | |
|---|---|
| Account | "Agentic" ••••8732 |
| Type | Limited margin, individual, self-directed |
| Starting capital | $500 (deposit pending at time of writing) |
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
chosen risk posture for a small account being run for entertainment, agreed 2026-09-08.

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
6. **Every order goes through `review_equity_order` and explicit human approval** before
   `place_equity_order`. No exceptions, no "obvious" trades.

## 4. Universe — built nightly, not fixed

No standing watchlist. Candidates come from the saved scanner:

- **Scan:** `Swing Momentum — Agentic`
- **`scan_id`:** `fbc9eab9-6206-4a2f-9de6-43dae608b450`

Filters:

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
candles. This is deliberate: a language model averaging numbers out of a JSON array is the
exact defect found in the prior n8n workflow.

| Check | Threshold | Tool |
|---|---|---|
| Trend | Close > SMA(20), daily | `type: sma, period: 20, interval: day` |
| Momentum | RSI(14) between **40 and 70** | `type: rsi, period: 14, interval: day` |
| Trend strength | ADX(14) > 20 | `type: adx, period: 14, interval: day` |
| Volatility | ATR(14) present | `type: atr, period: 14, interval: day` |
| Catalyst | News reviewed, no earnings ≤3 days | `get_equity_news`, `get_earnings_calendar` |

RSI is capped at 70, not 80: buying a name already extended into overbought gives the stop
no room and inverts the reward:risk.

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

## 6. Exits — checked every run

Exit on the **first** of:
1. Stop hit
2. Target hit
3. 5 trading days elapsed (time stop)
4. Thesis broken — the catalyst that justified entry is contradicted by news

Stops are managed by us on each run, not resting at the broker, so a gap through the stop
is exited at the next run's market price. This is a known limitation of a scheduled agent
and is the main argument against widening position size further.

## 7. Explicitly rejected

- **Crypto.** No historical bars and no indicators exist on the crypto tool surface —
  `get_crypto_quotes` returns bid/ask/mark and previous close only. Trading it would mean
  no moving average, no ATR, no scanner, and therefore no §5 at all.
- **Options.** Not approved on this account.
- **True day trading.** Requires continuous monitoring and per-order human approval;
  a scheduled agent can do neither. Considered and rejected 2026-09-08.

## 8. Decision log

Every run appends one JSON object per line to `decisions.jsonl`:

```json
{
  "ts": "2026-09-08T20:30:00Z",
  "run": "discovery|morning",
  "symbol": "OKLO",
  "action": "candidate|entry|hold|exit|skip",
  "reason": "RSI 47.4, close > SMA20, ADX 24",
  "entry": 43.59, "stop": 40.32, "target": 48.50,
  "shares": 5, "rr": 1.5,
  "indicators": {"sma20": 41.8, "rsi14": 47.4, "adx14": 24.1, "atr14": 2.89},
  "order_id": null, "outcome": null
}
```

`outcome` is backfilled on close. Scoring these is what tells us whether the strategy has
an edge — there is currently **no backtest and no evidence of one**.
