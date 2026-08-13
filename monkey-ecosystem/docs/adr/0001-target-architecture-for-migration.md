# Target architecture for the ecosystem migration

Status: proposed (target direction only — not a plan or timeline; see `CONTEXT-MAP.md` for the current state this is migrating away from)

Context: the ecosystem currently has at least three live pairs of old/new
service implementations sharing one gateway domain by path prefix
(`edu_app`/`edu_app_v2`, `edu_platform`/`edu_platform_go`,
`edu_story`/`edu_story_go`), with the old and new sides in each pair
reading/writing the *same* database tables — traffic is split between them
by whichever path each caller happens to use, not by any deliberate
cutover. See `CONTEXT-MAP.md` → Open Questions for the confirmed evidence.

Decided (2026-08-13), for later planning:

1. **Go over PHP.** Lean toward Go for new/rewritten services — either an
   existing Go repo or a new one — and phase out PHP.
2. **No shared data access.** Each service gets an isolated data store; no
   other service reads or writes it directly, not even read-only.
3. **Eliminate dual-write.** No service should write the same logical data
   into two stores/services as an integration mechanism.
4. **Lean toward EDA and event-sourcing.** Cross-service integration moves
   toward event-driven architecture with event-sourced write models, away
   from shared-database and synchronous direct-DB coupling.
