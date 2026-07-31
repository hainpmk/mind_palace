---
status: accepted
---

# Bootstrap discovery via automated cloud queries, kept separate from human-annotated component docs

Current infra is spread across 3 clouds with no up-to-date documentation, and some components aren't even identified yet. We decided to bootstrap the inventory with automated tooling against each cloud's API (AWS Config/Resource Explorer, Azure Resource Graph, GCP Cloud Asset Inventory) rather than authoring it from memory, so the starting point is ground truth. Automation output goes into `docs/components-raw/` as machine-generated snapshots, never directly into `docs/components/*.md` — the latter holds human-authored fields (criticality, owner, dependencies, open questions) that automated re-runs must not clobber. Re-running discovery means diffing the new raw snapshot against `docs/components/` to spot new/changed/removed resources (gap analysis), not overwriting it.
