---
paths:
  - "pine/**/*.pine"
---

# Pine Script v5 Conventions

- Target Pine Script v5 syntax only (`//@version=5`). Do not use v6-only syntax.
- Gate any "close-of-bar" logic behind `barstate.isconfirmed` wherever repaint
  risk exists (see non-repaint checklist in docs/02-technical-spec.md Section 5).
- Use `var` for state that persists across bars; never recompute persistent
  state from scratch each bar.
- Prefer Pine's user-defined types (`type ...`) for structured state (KeyLevel,
  CISDCandidate, FVGZone, TradeSetup) over loose parallel arrays.
- Comment each major pipeline section with a header banner, e.g.
  `// ===== SWEEP DETECTOR =====`, so the single file stays navigable.
- All session/level timestamps use explicit UTC — never rely on chart or
  exchange timezone defaults.
