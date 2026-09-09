# Agentic Swing Trading

A scheduled Claude agent that screens the market, analyses candidates, and logs every
decision for later scoring. It runs against the Robinhood "Agentic" account.

> **Currently in PAPER mode.** No real orders are placed. See `strategy.md` §0.
> A simulated order is never reported as a real one.

This repository is **public**. Keep account numbers, balances, credentials, and order IDs
out of it — the Routine prompts hold anything account-specific, and those are private.

## What this is, precisely

There is no application here. The system is three parts:

| Part | Where it lives |
|---|---|
| The spec (the "program") | `strategy.md` in this repo |
| The state (the "database") | `decisions.jsonl` in this repo |
| The runtime | Two scheduled Claude Routines, prompts held privately |
| The broker + market data | Robinhood's official MCP server |

No code, no dependencies, no credentials. Authentication is handled by the Robinhood
connector; nothing in this repo touches it.

## Files

| File | Purpose |
|---|---|
| `strategy.md` | Governing spec — execution mode, risk envelope, hard limits, entry/exit rules, log schema |
| `decisions.jsonl` | Append-only log, one JSON object per decision |

## The two scheduled runs

**Discovery — 4:30 PM ET, weekdays.** Runs the saved scanner against the completed session,
computes indicators, checks news and earnings, and writes tomorrow's ranked candidates.
Places nothing. Cannot check spread — the order book is empty outside market hours.

**Morning — 9:45 AM ET, weekdays.** Reviews open positions against the exit rules first,
then re-confirms yesterday's candidates on live data, checks spread and tradability,
re-sizes at the actual limit price, and proposes orders.

Deliberately 9:45 and not at the open: pre-market and the first minutes of the session have
structurally wide spreads that say nothing about the stock. Measured 2026-09-09 at 09:14 ET,
four of five candidates showed spreads of 1.2–9.5% that would have failed the 0.5% gate for
the time of day alone.

Exits are evaluated before entries. Freeing a slot matters more than filling one.

## Scanner

Saved on the Robinhood account as **Swing Momentum — Agentic**
(`scan_id: fbc9eab9-6206-4a2f-9de6-43dae608b450`). Filters documented in `strategy.md` §4.

## Reading the log

```bash
# today's candidates
grep '"action": "candidate"' decisions.jsonl | tail -20

# every closed decision with its outcome
grep -v '"outcome": null' decisions.jsonl

# anything skipped for stale data or no session
grep -E '"action": "(no-session|skip)"' decisions.jsonl | tail
```

## Known limitations — read before trusting this

- **No backtest. No evidence of edge.** PAPER mode exists to find out whether there is one.
  20 scored decisions is the bar before considering live.
- **Stops are not resting at the broker.** They are enforced when a run fires, so a gap
  between runs is exited at the next run's price. Broker alerts give notification, not
  protection.
- **Scheduled, not continuous.** Between runs, nothing is watching.
- **No live trading calendar.** `trading://market-hours` is a static clock reference and
  does not know about holidays or half-days; the runs infer the session from quote
  freshness instead (§3.9).
- **Crontabs are UTC**, set for Eastern Daylight Time. They drift one hour when EST begins
  in November and need re-pointing then.
- **`review_equity_order` returning no alerts does not mean an order is valid.** Verified
  2026-09-08. Funding is checked explicitly per §3.7.
