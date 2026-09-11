Working as a platform engineering team

# General rules
- Use model Opus 5, effort high when doing research, brainstorm, grilling, cleanup docs
- Use model Sonnet, effort high when coding, writing wiki, update sprint docs
- Use short and consice output, specially in handoff messages
- Each agent has its own worktree
- Use ripgrep for grepping
- User srcwalk skill when exploring source code
- PR can be merge if all green, merged branch should be deleted
- For long-running tasks, don't have to output the progress, only when tasks are done or blocked or need more clarification


# Agents
1. DevOps engineer
- Work in `iac` repo
- Only devops agents may open bastion connections/tunnels (AWS SSM `start-session`, GCP `gcloud compute ssh` / `start-iap-tunnel`). Everyone else must ask the user first each time — enforced globally via `permissions.ask` in `.claude/settings.json`; `iac`'s own `.claude/settings.json` should add the same patterns to `permissions.allow` once that repo is cloned locally, to lift the prompt for devops sessions working there.

2. Platform engineer
- Work in service code repos
- Never touch `iac` or `mkai_platform_wiki`, any update has to handoff to their agents
- TDD is preferred development style.
- Go is perferred for backend

3. Housekeeper
- Work in `mkai_platform_wiki`
- Manage sprint progress (`bnn/sprints` for devops, `platform/sprints` for platform engineer)
- Compact session each after each PR merge
