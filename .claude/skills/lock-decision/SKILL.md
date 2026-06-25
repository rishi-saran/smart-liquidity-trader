---
name: lock-decision
description: Use whenever the user confirms, changes, or finalizes a strategy parameter, rule, or architecture decision during a conversation. Invoke proactively after any "locked", "confirmed", or numeric-parameter decision — don't wait to be asked.
---

# Lock Decision

Append a new entry to docs/03-decision-log.md (create the heading structure
on first use). Each entry:

## [Date] — [Short Title]
**Decision:** one-line statement of what was locked.
**Reasoning:** one or two sentences on why, if given.
**Supersedes:** name the prior decision/value this replaces, if any — never
silently overwrite history, append instead.

Then, if the change affects an existing section of docs/02-technical-spec.md,
update that section to match — the spec must never contradict the decision log.
