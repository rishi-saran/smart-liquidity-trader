---
name: repaint-audit
description: Use when a Pine Script module (level, sweep, CISD, FVG, or trade logic) is reported as complete and needs a non-repaint check before being marked done. Also invoke when the user asks "does this repaint" or "check for repainting".
---

# Repaint Audit

Walk through the module being checked against this list (from
docs/02-technical-spec.md Section 5):

1. Does any "current day" level (PDH/PDL, Asia H/L, VP POC/VAH/VAL) ever read
   from a still-forming period instead of a finalized prior one?
2. Does any close-price check fire before `barstate.isconfirmed` is true?
3. Does the 14-bar displacement average ever include the current forming bar?
4. Is any FVG zone instantiated before candle 3 has actually closed?
5. Would this logic produce a different signal on historical replay vs.
   real-time bar-by-bar progression? Reason through this explicitly.

Report findings as pass/fail per item, not just an overall verdict. If
anything fails, propose the minimal fix — don't silently rewrite unrelated logic.
