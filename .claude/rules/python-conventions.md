---
paths:
  - "analysis/**/*.py"
---

# Python Analysis Layer Conventions

- Keep scripts dependency-light: pandas, numpy, matplotlib only unless the
  user approves adding something else.
- Scripts run standalone via `python scripts/<name>.py <csv-path>` — no
  notebook-only logic for anything meant to be reusable.
- Never hardcode a CSV path; always accept it as an argument.
- Trade CSVs are exports from TradingView's Strategy Tester — assume columns
  may need normalization and validate before computing stats.
