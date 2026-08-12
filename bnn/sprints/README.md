# Sprints

Lightweight task tracking for `bnn`/`iac` work, separate from `docs/` (which documents *state* — what exists, what's decided) and `docs/adr/` (which documents *decisions*). This tracks *work still to do*.

- **`backlog.md`** — every outstanding task, not yet prioritized into a specific work cycle. Add to it whenever new work is identified (a "Not yet done" note in a component doc, a follow-up from a migration, a new initiative) — this is the single place all of that should end up, not scattered across component docs indefinitely.
- **`sprint-N.md`** — a small, curated pull from the backlog: what's actually being focused on right now. Not every backlog item needs to be in a sprint. When a sprint's tasks are done, start the next `sprint-N.md`; don't reuse/extend an old one.

Task status markers: `[ ]` open, `[x]` done, `[~]` in progress, `[!]` blocked (note on what).
