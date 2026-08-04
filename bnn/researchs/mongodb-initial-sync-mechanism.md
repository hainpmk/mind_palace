# MongoDB initial sync: why it's a full re-clone, not an incremental/hash-diffed transfer

Context: observed directly during the `vm-mongo-master` → AWS migration (see `docs/components/vm-mongo-master-migration-postmortem.md`). A transient network drop between the Azure source and the AWS secondary caused the entire initial sync to restart from zero, discarding ~12 hours of progress. This note captures why, since the mechanism is less obvious than it first seems and is worth understanding before relying on it again (e.g. for the deferred `vm-core-database` migration).

## Two different replication mechanisms, easy to conflate

- **Steady-state replication (oplog tailing)** — genuinely incremental. Once a node is caught up, replica members stream the oplog (an append-only log of write operations) and apply each operation in order. This is the "small incremental units" model.
- **Initial sync** (what runs when a brand-new member joins) — a full collection copy via a live cursor, not a chunk-diff. It opens a `find()` cursor against each collection on the source at one consistent snapshot point, streams every document across, then replays the oplog for whatever changed while that took. Nothing is compared/hashed against the destination — there's nothing to diff against, since the destination starts empty.

## Why hash/chunk-based reconciliation (rsync-style) doesn't apply

Content-addressed diffing (rsync, and similar chunk-hash strategies) is a tool for syncing two datasets that already substantially overlap — it finds the delta so you don't re-transfer what's already there. Initial sync's starting condition is a fully empty destination, so there is no "already there" to diff against; the entire dataset must be transferred regardless. The intuition ("shouldn't it hash each chunk?") is reasonable for general data-sync problems, just doesn't match this specific scenario.

## Why a dropped connection can't gracefully resume mid-clone

The whole clone operation is pinned to one consistent point-in-time snapshot on the source (via WiredTiger's MVCC snapshotting). If the connection carrying that snapshot's cursor dies:
- There's no cheap way to "reattach" at the exact same logical position while still guaranteeing consistency.
- Documents may have changed on the source in the meantime, so any partially-copied data can't be trusted without re-verifying it — which is most of the cost of just re-cloning it anyway.

## Resumable initial sync exists, but has real limits

MongoDB added a "resumable initial sync" feature (since 4.4) covering *some* failure classes — brief network blips during specific operations can sometimes reconnect and continue without a full restart. It is not universal. In the observed failure, the error was:

```
InitialSyncFailure: HostUnreachable: Error cloning collection 'edu_lesson.process_checkpoints'
:: caused by :: network error while attempting to run command 'collStats' on host '...'
```

An entire host becoming unreachable (not just one cursor briefly hiccuping) fell outside whatever resumability covers, and triggered a hard restart from `initialSyncAttempt: 1` to `initialSyncAttempt: 2` (of a `initialSyncMaxAttempts: 10` ceiling) — confirmed by the fact that attempt 2's collection-clone progress built up from nothing over several more hours, none of attempt 1's ~3TB (logical) already-copied data carried over.

## Practical implication for cross-cloud/over-the-internet initial syncs

This mechanism is designed for the common case: replica set members on the same LAN/VPC with a stable, low-latency link. It's a poor fit for a migration bridge running over the public internet with a temporary/unreliable path — any connectivity blip doesn't just pause progress, it can erase the entire elapsed attempt. Worth factoring into future cross-cloud migrations: either budget for the possibility of a full restart (as happened here), or invest in a more stable dedicated path (VPN/peering) specifically to protect a long-running initial sync, rather than a raw public-IP bridge.

## Open questions / further reading (not yet researched)

- Exact scope of what MongoDB's resumable-initial-sync feature does and doesn't cover (which error classes, what time/state budget) — would clarify whether a different network setup could have avoided the restart, or if this specific failure mode (host unreachable) is never resumable regardless.
- Whether `mongomirror`/Atlas Live Migration (mentioned earlier in the mongodb.md discovery as an alternative migration path) uses a different, more resilient mechanism than raw replica-set-join initial sync — relevant if a similar migration needs redoing.
- Whether there's a way to reduce a single collection's clone risk window (e.g., splitting an extremely large collection like `edu_backend.history_user_action` at ~412GB into a separate, checkpointed transfer) rather than relying on the all-or-nothing initial sync for the whole dataset.
