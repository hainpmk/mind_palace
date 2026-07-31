bnn - an IaC repo
===

bnn is short for banana, is home for my whole system Infras-as-Code (Terraform).

# Purpose
Build infras that:
- is a centralized place for IaC, using Terraform
- has semi-automated workflow (engineers and agents with devops role can operate)
- is safely reproducible
- is safely testable
- can be monitored and observed 24/7
- is secured
- gradually manage components through code without breaking current system

# Current:

1. infras is deployed on 3 clouds (AWS, GCP, Azure), and should be migrated into AWS only
2. no up-to-date docs describe what system looks like or how components interact
3. some components has been coded (or manage with code - Helm) (eg: some service on eks, argocd, jenkin CI)
4. some components are manually setup (eg: vm, ec2 nodes, eks nodepool, ...)
5. mongodb is on Azure, run on a vm but not sure in standalone or cluster mode
6. some AI workloads are on Azure
7. some not-yet-identified components are on GCP

# Glossary

**Adopt-on-touch**: the policy for bringing an existing, manually-managed infrastructure component under Terraform management. A component is imported into Terraform only at the moment someone needs to change it for a real reason (resize, config change, add a nodepool, etc.) — never as a standalone migration project. Discovery/inventory of existing components is a separate, continuous activity and does not by itself trigger import.

**Environment tier**: the deployment classification of a workload — `dev`, `stg`, or `live`. This is a per-app classification, not a uniform ladder every domain must climb: some apps have no `stg` tier at all and go straight from `dev` to `live`. On the AWS target architecture, each tier is a separate cluster/VPC (isolating blast radius), not namespaces within one shared cluster.

**Lifecycle state**: an application-level flag/state that a live workload uses to distinguish test-like behavior from real behavior — e.g. a payment's `is_sandbox` flag (Apple/Google App Store notifications only ever hit the one live webhook, there is no separate sandbox endpoint), or a content item's draft-vs-published state (creators view/test draft material through the live app, gated by role, before it's flagged published for end-users). This is orthogonal to **environment tier**: it's gated by application logic/auth inside `live`, not by infrastructure/network boundaries between environments. Do not conflate a "stg" need with lifecycle state — most cases described as "testing against live" turn out to be this, not a real stg-to-live network requirement.
