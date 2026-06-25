# Smart Liquidity Trader V1

Pine Script v5 trading system for Gold (XAUUSD) and NAS100 — a highly selective
liquidity-sweep reversal strategy.

Full philosophy: @docs/01-philosophy.md
Full architecture: @docs/02-technical-spec.md
Every locked decision, in order: @docs/03-decision-log.md

## Stack (V1 scope only)
- Pine Script v5 — the entire system lives in ONE file: `pine/smart_liquidity_trader_v1.pine`
- Python (pandas/numpy/matplotlib) for post-backtest statistical analysis — `analysis/`
- NO backend, NO database, NO frontend in V1. Do not propose adding them — that's a V2+ concern.

## Critical workflow fact
TradingView has no API to push or deploy Pine code. The only way to test a script
is: edit the file here → user manually copies the contents → pastes into
TradingView's web Pine Editor → clicks "Add to chart". Never suggest a CLI or
automation shortcut for this step — it does not exist.

## Hard rules (non-negotiable)
- Non-repainting is absolute. Any value depending on an unclosed bar is a bug,
  not a stylistic choice. Run the repaint-audit skill on a module before
  marking it done.
- Every strategy rule must be objective (TRUE/FALSE). Never introduce a
  subjective condition, even as a "temporary" placeholder.
- Never change a locked parameter (e.g. "1.5x displacement", "14-bar lookback",
  "10pt/50pt SL cap") without the user explicitly confirming the change. Check
  docs/03-decision-log.md if unsure whether something is locked.

## Build/verify commands
- Pine: no build step — verification happens after the user pastes into
  TradingView and reports back what they saw.
- Python: `cd analysis && pip install -r requirements.txt`, then run scripts
  directly, e.g. `python scripts/stats_report.py path/to/export.csv`

## Folder map
- `pine/` — the strategy/indicator script (single file, no splitting)
- `analysis/` — Python stats layer, reads CSVs exported from TradingView's
  Strategy Tester, dropped into `analysis/data/` (gitignored)
- `docs/` — philosophy, technical spec, decision log — source of truth; read
  before making any architecture or parameter decision

## Workflow
- Build the Pine script ONE pipeline module at a time, in order: Session Engine
  → Level Engine → Sweep Detector → CISD Engine → FVG Engine → Trade Builder →
  Trade Manager → Visualization. Do not start a later module before the current
  one is visually verified by the user on a live chart.
- Commit after each module is confirmed working, not before. One module per commit.
- When a new parameter or rule gets decided in conversation, use the
  lock-decision skill to record it — don't just act on it silently.
