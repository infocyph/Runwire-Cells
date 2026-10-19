# Runwire Cells — independent Rust stateful PHP compute: full draft plan

**Status:** DRAFT / architecture and planning only
**Updated:** 2026-10-19
**Base:** `main` (license-only starter repository) @ `184421f7c6748acd7fe77d62eefdf61809ac6151`
**Branch:** `plan/runwire-independent-ecosystem-roadmaps`
**Implementation:** NOT STARTED
**Publication:** No release, tag or merge authorized.

## Vision

A standalone Rust-backed, PHP-facing, stateful actor/cell engine: activate addressable PHP objects on demand, process bounded mailbox messages, persist recoverable state, and ultimately place and recover cells across independent nodes.

**Complete product independence:** Does not depend on `infocyph/runwire` or `infocyph/runwire-core`, cannot require their PHP extensions, must have its own native executor, public PHP API, tests, benchmarks, packaging and release cadence. Related branding does not imply API compatibility or shared scheduler. Any external storage/service dependencies must be specified explicitly, not hidden.

Inspired by the *problem domain* of celld/Durable Objects, **not** a promise of Cloudflare Worker API compatibility, V8 execution, SQLite replication equivalence or automatic drop-in replacement.

## Architectural truth / key decision

A PIE PHP extension is a distribution/integration mechanism, **not automatically a durable multi-node service**. A PHP user-defined object cannot be run on arbitrary Rust worker threads or migrated as a live Zend object between machines.

Separate concerns:
1. **PIE extension**: PHP developer APIs, local valid Zend context, request/response bridge, serialization boundary.
2. **Persistent cell host**: managed PHP worker processes and Rust native coordinator/executor, keeping activated PHP object lifetimes valid.
3. **Distributed control/data plane** (later): cluster routing, exclusive ownership/fencing, durable recovery and peer communication. A native daemon/binary may be needed and must be justified and packaged as part of the Cells product, never by depending on Runwire Core.

**Do not commit to a deployment architecture until the PHP-host proof of concept and crash-safety storage model are validated.**

## Core semantics: proposed contract

- Each cell has stable identity: type + canonical key; deterministic routing and authorization.
- A cell has one **active writer** within the selected failure domain. Its mailbox has explicit capacity, ordering, overflow and deadline policy.
- Sequential mutation **within a cell**; independent cells may run concurrently across host PHP workers.
- Activation: reconstruct PHP object and durable state; hibernation/eviction: release PHP VM object and resources; no one-Fiber-per-identity global resident assumption.
- Persistence: state mutations with well-defined transaction boundary, response acknowledgment durability and failure semantics.
- Message classes: synchronous request/reply vs durable asynchronous messages; do not conflate their acknowledgement guarantees.
- Idempotency keys and replay/retry semantics are explicit; no blanket exactly-once claim.
- Backpressure and quotas per cell, tenant (if introduced), PHP worker and node; anti-hot-cell overload policy.
- Cell-level error isolates its mailbox/instance where safe; host death triggers explicit recovery.
- Cancellation/deadline propagation must not corrupt atomic state updates. External side effects need explicit outbox/idempotency story.
- Public contract must distinguish live state, snapshot state, committed durable state and durable acknowledgments.

## Independence, trust and data safety

- No required dependency on PHP Runwire or Rust Runwire-Core, no shared PHP extension ABI or mandatory runtime symbols.
- Do not build distributed correctness atop time-based lock expiration alone. Fencing epochs must be enforced by the authoritative write path.
- Exactly one active owner may execute accepted mutations; when ownership cannot be proven, **fail closed** instead of serving writes on multiple nodes.
- Network partitions, bucket unavailability, stalled RPCs and clock drift are first-class fault models.
- No unqualified RPO=0, linearizability, or exactly-once statement: each proposed mode gets a separately tested guarantee matrix.
- Untrusted callers require explicit authentication, per-identity authorization and admission; service-to-service transport must be authenticated/encrypted in production.
- Persisted state and messages must use versioned serialization, migration rules, checksum/integrity and recoverable partial-write handling.

## Proposed components (boundaries provisional)

```
Runwire-Cells
├── rust/
│   ├── cell-logic/          # state machine, ordering, decisions; deterministic tests
│   ├── cell-runtime/        # activation, mailbox and worker/host ownership
│   ├── cell-storage/        # transaction, snapshot, durability adapters
│   ├── cell-protocol/       # versioned RPC envelopes, error and auth semantics
│   ├── cell-cluster/        # post-MVP ownership, fencing, rebalancing/recovery
│   └── php-extension/       # Zend API, PHP classes, PIE integration
├── host/                   # optional independent CLI host/daemon packaging
├── php/                    # PHP stubs and independent integration fixtures
├── tests/                  # Rust, PHP, fault injection, multi-node
├── benchmarks/             # warm/cold/hibernated/hot-cell/failover
└── docs/plans/
```

## PHP API exploration — NOT committed implementation

```php
<?php
// Illustrative only; concrete namespaces and signatures require PHP-VM feasibility review.

$room = \RunwireCells\Cells::get(Room::class, 'room:42');
$reply = $room->call('join', ['user-123'], idempotencyKey: 'request-abc');
```

Candidate cell interface: lifecycle callbacks `onActivate`, `onMessage`, `onDeactivate`, transactions through a context-managed state object. Avoid unrestricted reflection-driven arbitrary public method invocation without an authorization/allowlist model. Do not use PHP serializing raw closures, resources or arbitrary live objects as durable state.

## Storage and durability decision record (must precede cluster work)

Evaluate two or more explicit backends in local MVP:
- SQLite embedded per host/cell (carefully defined process/lock/transaction ownership), or
- transactional external database suitable for enforced ownership fencing and atomic durable writes.

Comparison criteria: sustained writes, fsync/durability cost, state recovery, write amplification, snapshot latency, failure-injection behavior, backup integrity, replication suitability, operational complexity. Do not assume that SQLite-per-cell and object-store LTX replication can be imported without their own correctness proof.

Required semantics:
- Atomic state transition and durable message acknowledgment.
- Duplicate RPC/replay handling through stable operation IDs and durable deduplication window.
- Corruption detection and deterministic recovery from last committed state.
- State schema evolution/migration and storage-version compatibility.
- Exact meaning of `acknowledged`, `accepted`, `committed`, and `delivered`.

## Program batches / tracker

| Batch | Deliverable | Exit criteria | Status |
|---|---|---|---|
| S0 | PHP host + PIE feasibility and requirements | PHP state can safely live across requests in managed workers, restore after crash, clean unload | [ ] Not started |
| S1 | Local identity, mailbox and scheduler | Serialized per-cell mutations, bounded queues, independent cell fairness | [ ] Not started |
| S2 | Transactional durable state | Crash/restart and committed-vs-acknowledged proofs under faults | [ ] Not started |
| S3 | Activation, eviction, timers and resource accounting | Cold/warm/hibernated benchmarks, bounded resident footprint | [ ] Not started |
| S4 | Local production MVP | Auth, quotas, safe errors, observability, chaos/soak, PIE packaging | [ ] Not started |
| S5 | Multi-process local placement and IPC | Worker death, state reactivation, no split-brain writer | [ ] Not started |
| S6 | Cluster transport, routing and membership | Authenticated RPC, bounded queues, peer reconnect, invalid routing | [ ] Not started |
| S7 | Exclusive ownership, leases and fencing | Simultaneous failure, clock skew, partition, slow store/no stale writes | [ ] Not started |
| S8 | Distributed durability and recovery | Recovery guarantee matrix with measured RPO/RTO under node/storage faults | [ ] Not started |
| S9 | Rebalancing, rolling upgrades and operations | Drain/reassign, state migration, schema/version management, backups | [ ] Not started |
| S10 | Independent release certification | Exact source/artifacts, multi-OS/PHP matrix, performance and fault evidence | [ ] Not started |

### S0 — lifecycle feasibility gate

1. Implement trivial PIE extension `runwire_cells`, expose only a safe identity/echo API in PHP.
2. Prove managed persistent PHP worker process with process-correct Zend initialization, object creation, activation and destruction.
3. Test cancellation/exception during message execution; queue lifecycle independent of RINIT/RSHUTDOWN.
4. Demonstrate restart restoring serialized state; never transport live zvals across threads or nodes.
5. Compare in-process actor execution vs IPC host design and select an ownership model using CPU/RSS/latency and security evidence.
6. Document how PIE installation deploys any persistent service executable, if required. Installation must not silently create insecure root services.

### S1–S3 — local correctness before distribution

- Identity canonicalization, placement on local PHP executors, bounded mailbox, serial message processing and fairness across cells.
- Explicit overload responses, sender timeouts, queue depth and per-cell starvation monitoring.
- Define transaction boundary, data model, write contention and durable acknowledgment; exercise kill -9, power-loss simulation where feasible and corruption cases.
- Activation cache policy, idle timers and eviction without dropping outstanding acknowledged work.
- Rate/CPU/RSS under 1, 100, 10k and 100k *known* identities; distinguish known identities from concurrently active cells.

### S4–S6 — operational local and multi-node interfaces

- Authentication and authorization between PHP caller and cell host, service-to-service authorization, configuration, secrets and TLS.
- Versioned RPC protocol; stable operation IDs, bounded payloads, timeouts, idempotency, safe redaction.
- Multi-process local placement with restart and worker generation; recover from process crash without duplicate active writer.
- Cluster gateway/routing should not require an external load balancer for correctness; external LB may still terminate TLS/public traffic.

### S7–S9 — consensus-sensitive lifecycle

- Explicitly select consistency model (single authority, conditional-write store, or consensus control plane) and prove owner fencing.
- Partition tests: stale writer isolated from peer, lease expires while paused, ownership moves, old owner resumes. Storage must reject its writes.
- Extend to rolling replacement, cross-version RPC, migration/schema evolution, backup restore, member replacement.
- Publish honest tradeoff matrix for availability vs consistency, recovery times and durability modes; never copy celld benchmark numbers as our goals without matched tests.

## Benchmark plan

Measure: warm call p50/p95/p99; cold activate p50/p99; hibernated wake; resident bytes per active cell; bytes per *known* identity; durable write commit latency; throughput per hot cell vs many independent cells; message backlog/fairness; node CPU/RSS; failover RTO and acknowledged write loss RPO.

Client types: local PHP extension client, remote PHP client, independent RPC implementation for protocol interop. Workloads: counters, transactional account-like object (not production ledger claim), chat room fanout, timers, replay/idempotency, varying state size, hot-key overload and many-idle-cell footprint.

Competitor comparisons: optional celld/Durable Objects-style system **only** for equivalent durability mode, placement, access path, workload, topology and failure model. Report hardware, schema, network latency, storage latency, operations, all raw records; no blanket "faster than celld".

## QA and security gates

- `cargo fmt --check`, Clippy, audit, Rust unit/property/fuzz and deterministic state-machine simulation.
- PHP 8.4/8.5 extension install and clean lifecycle on NTS/ZTS variants; Valgrind/ASan where feasible for ABI boundary.
- Kill -9 during commit, lost ACK after commit, duplicate delivery, slow/failing storage, corrupted snapshot, partial write, hot mailbox pressure, deadlock, PHP callback exception and exhaustion.
- Multi-node tests for partitions, delayed peer messages, stale ownership generation, reordered RPC, clock jumps, rolling restarts and backup restore.
- Enforce every RPC payload limit, backpressure, TLS/auth and per-identity permissions; do not assume users of a shared cluster are trusted.
- Capture provenance by exact commit and artifact, including skipped/cancelled tests.

## Evidence template (mandatory every batch)

| Field | Requirement |
|---|---|
| Batch | Sn |
| Commit | Exact source SHA and dependencies |
| Verified behavior | Contract/assertions and log artifacts |
| Failure cases | Reproduction, recovery trace and invariant checks |
| Benchmark | Raw records, duration, percentile, CPU/RSS, disk/network |
| Storage modes | Exactly which durability mode tested |
| CI | Exact run IDs, no skipped gate disguised as pass |
| Known gaps | Severity and next phase |
| Tracker | Update in same commit as evidence |

## Open decisions to resolve in S0 / before S2

1. Does Cells ship an embedded persistent PHP worker host, a managed CLI service, or a standalone daemon with PHP clients? Distinguish installed extension and deployed service.
2. Who owns the PHP VM per active cell, and when does PHP object lifetime end?
3. Local durable storage choice and transaction/ACK model.
4. Security tenancy boundary: single application trusted workload at MVP, or hostile multi-tenant support later? Do not suggest multi-tenancy before sandbox isolation.
5. Is a standard DB backend adequate for cluster fencing, or is a dedicated conditional-write/consensus store needed?
6. What recovery guarantees are minimum release blockers, and which deployment topology supports them?
7. Data schema evolution, backup, cross-version messages and rolling upgrades.

**This planning branch is documentation-only. No Rust implementation, implicit dependencies or durability guarantees are asserted.**
