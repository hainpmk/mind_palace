---
status: accepted
---

# Two clusters (nonprod, live) with independent per-cluster ArgoCD, not shared cross-cluster tooling

The AWS target topology for Kubernetes workloads is two clusters: `nonprod` (hosts all dev and stg workloads — `stg` is a per-app **lifecycle tier**, not every app has one) and `live`. We considered a single shared cluster namespaced by environment tier, and rejected it: a live workload shouldn't compete for nodes/control-plane with dev/stg, and a bad nonprod deploy shouldn't be able to affect live's availability.

For GitOps tooling (ArgoCD) that would otherwise span both clusters, we run **independent ArgoCD instances per cluster** rather than one shared instance (or a third dedicated management cluster) holding deploy credentials to both. A single cross-cluster ArgoCD is a blast-radius regression on the same reasoning that justified splitting the clusters: if nonprod is compromised, an ArgoCD instance with live-reaching credentials becomes the pivot path straight into live. The cost — losing a single pane of glass, duplicated GitOps repo config — is acceptable at current scale (2 clusters). Revisit only if a real, felt need for a shared view emerges (adopt-on-touch reasoning), not preemptively.
