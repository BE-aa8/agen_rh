# Agentic Swing Trading

A scheduled Claude Code agent that screens the market, proposes swing trades on the
Robinhood "Agentic" account, and logs every decision for later scoring.

**Nothing here trades autonomously.** Each order is simulated with `review_equity_order`,
presented with its cost and pre-trade alerts, and placed only after explicit human
approval.

This repository is **public**. Keep account numbers, balances, credentials, and order IDs
out of it — the Routine prompts hold anything account-specific, and those are private.

## Files

| File | Purpose |
|---|---|
| `strategy.md` | The governing spec — risk envelope, entry/exit rules, hard limits |
| `decisions.jsonl` | Append-only log, one JSON object per decision |

## The two scheduled runs

**Discovery — 4:30 PM ET, weekdays.** Runs the saved scanner against the completed
session, checks news and the earnings calendar, computes indicators on the survivors, and
writes tomorrow's ranked candidates to the log.

**Morning — 9:00 AM ET, weekdays.** Reviews open positions against the exit rules first,
then re-confirms yesterday's candidates on pre-market, sizes them, and brings orders for
approval.

Exits are evaluated before entries. Freeing a slot matters more than filling one.

## Scanner

Saved on the Robinhood account as **Swing Momentum — Agentic**
(`scan_id: fbc9eab9-6206-4a2f-9de6-43dae608b450`). Filters are documented in
`strategy.md` §4. Re-run it with `run_scan`; it evaluates against live market data each time.

## Reading the log

```bash
# today's candidates
grep '"action":"candidate"' decisions.jsonl | tail -20

# every closed trade with its outcome
grep -v '"outcome":null' decisions.jsonl
```

## Known limitations

- **No backtest.** There is no evidence this strategy has an edge. The log exists to find out.
- **Stops are not resting at the broker.** They are enforced on each scheduled run, so a
  gap through a stop is exited at the next run's price, not the stop price.
- **Scheduled, not continuous.** Between runs, nothing is watching.
- **Crontabs are UTC.** The schedules are set for Eastern Daylight Time and will drift one
  hour when EST begins in November — they need re-pointing then.
