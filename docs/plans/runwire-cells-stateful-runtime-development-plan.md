# Runwire Cells — independent Rust stateful PHP compute: full draft plan

**Status:** DRAFT / detailed implementation plan; all tasks open
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

## Detailed implementation specification and task-level tracker

The S0–S10 roadmap above is the high-level progress summary. The numbered checkboxes below are the **canonical work-item tracker**. Since this repository starts with only LICENSE and this plan, all paths and APIs are proposed. They are not evidence of implemented functionality, proven consistency, completed QA, or production readiness.

### Product decision and architecture boundaries

- **Confirmed:** Runwire-Cells is its own Rust-first, PHP-facing actor/cell execution product; no required dependencies on infocyph/runwire or infocyph/runwire-core, no shared PHP extension ABI, no automatic integration or identical API requirement. Its release process is independent.
- **Confirmed:** PIE is the target installation mechanism for the PHP extension. PHP 8.4/8.5 are the initial feasibility targets, with explicit validation by PHP ABI, NTS/ZTS, and architecture.
- **Not fixed before S0:** how the PHP worker host is deployed, which component owns active PHP objects, state backend, mailbox durability options, control-plane consensus mechanism or native daemon packaging. These are design gates, not details to assume.
- **Critical boundary:** Rust networking and storage tasks can run on Rust executors, but PHP objects, zvals, class methods and callbacks may execute **only on their valid owning PHP Zend VM/request/worker thread**. Moving a cell to another host means serializing committed state and reactivating a new PHP instance, not migrating a live object.
- **No false guarantees:** serial local mailbox execution does not imply cross-node consistency. Lease expiration without authoritative write fencing cannot guarantee single writer. Durable commit does not guarantee exactly-once external effects. Availability under network partitions is an explicit policy.
- **MVP trust boundary:** one trusted application/deployment until auth and isolation requirements are proven; hostile multi-tenancy, arbitrary untrusted PHP class execution and code sandboxing are **not supported claims**.
- **Node decomposition under evaluation:** PHP extension API (PIE), persistent PHP cell workers/host and independent Rust controller/RPC service where warranted. If PIE does not deliver/manage the standalone host executable safely, document distinct installation and service startup steps. Never auto-launch an insecure root daemon.
- **Architectural aim:** very low warm-call overhead, efficient idle-cell memory footprint, bounded queueing, independently concurrent cells, durable storage cost transparency and robust recovery. Measure full PHP user-method execution, not just a Rust actor microbenchmark.

### Core semantic invariants to lock in ADRs

1. **Identity:** a canonical cell key consists of immutable cell-type identifier, version-safe namespace and validated instance ID. No hidden collisions, path traversal, cross-tenant identity leakage or ambiguous UTF-8 normalization.
2. **Execution:** one active mutation handler per cell; within-cell ordering is explicit, FIFO under a specified admission policy, and unrelated cells make progress independently.
3. **Calls:** differentiate synchronous RPC reply from asynchronous durable enqueue; define accept, schedule, execute, commit, acknowledge, retry and duplicate status as separate states.
4. **Transaction:** committed state and durable acknowledgement share an atomic authority wherever required. A state commit followed by lost response must be safely recognized on retry using operation IDs and bounded durable deduplication.
5. **Cancellation:** cancelling the *caller* is not proof that an already committed mutation was cancelled. Define cancellation-before-admission, during waiting, during execution, and after commit separately.
6. **Activation/eviction:** a cell's PHP instance can be discarded only after safe quiescence or after fencing/aborting work according to semantics. Rehydrate from the last valid durable checkpoint plus any committed log if a log is used.
7. **Ownership:** a node can execute mutating cell work only with validated generation/epoch, and the authoritative durable write path enforces fencing. An expired owner must not commit if it resumes.
8. **Recovery:** choose and document durability mode, storage guarantees and failure domain. Make RPO/RTO claims only for a specific tested topology/failure model.
9. **External effects:** no implicit distributed transaction with network APIs; require outbox/idempotency or developer-facing warnings and contracts for nontransactional effects.
10. **Security:** authenticate client-to-host and node-to-node traffic for any networked deployment; authorize cell type, identity and allowed operation before dispatch; never invoke arbitrary PHP methods from raw remote strings.

### Suggested source layout and independently testable modules

- rust/cell-logic/: deterministic core state machines for mailbox transitions, epochs, ownership, errors, durable-operation decisions; keep pure logic independent from wall clock/network.
- rust/cell-runtime/: local actor registry, PHP host lifecycle coordination, cell activation, bounded per-cell/global queues, fairness and idle eviction.
- rust/cell-storage/: state codec, transactional storage ports, local implementation(s), migration/snapshot/replay, idempotency and storage failure handling.
- rust/cell-protocol/: versioned envelopes, cell IDs, error/status codes, idempotency headers, auth and payload limits; only build transport code after local correctness.
- rust/cell-cluster/: membership, routing, fencing and takeover, recovery, failover and anti-stale message handling (post-MVP).
- rust/php-extension/: Zend/PIE binding, safe PHP classes, executor dispatch, exception mapping, request lifecycle integration.
- host/: if S0 proves a companion CLI executable/service is necessary, build it **inside this product** with explicit package/install/run instructions.
- php/stubs/ and tests/php/: user API stubs, examples, behavior and PHP ABI safety tests.
- tests/simulation/, tests/chaos/, tests/integration/: deterministic concurrency tests, local process restarts, real multi-node storage outages/partitions.
- benchmarks/: cold activate/hibernated wake, warm calls, durable commits, single hot cell, many parallel cells, memory per resident and known identity, failure recovery.
- docs/adr/ and docs/plans/evidence/: approved choices, exact commit/CI/environment and raw evidence reference manifests.

Keep crates/abstractions only where they reduce complexity, isolate a safety boundary or support deterministic testing. Avoid premature plugin systems, framework dependencies or optional features in the hot path.

### Required workflow for every batch

1. Confirm last approved S(n-1) evidence and current SHA; update the plan before any change to scope, durability guarantees or safety model.
2. Isolate the smallest vertical slice: executable PHP call, native coordination, storage effect (when needed), failure behavior and observability.
3. Test Rust logic/property/model behavior, Zend/PIE lifecycle, real PHP workers, crash/process faults, security/limits and exact storage semantics.
4. Run representative repeated benchmarks: successful committed operations per second, warm/activation p50/p95/p99, CPU, RSS, queue depth, total acknowledged data lost and recovery time.
5. Prefer correctness and predictable operations to optimistic speed claims. No benchmark result is valid if it excludes crash recovery or required durability when comparing a durable mode.
6. Every task/entire batch remains unchecked until code, tests, CI results, appropriate adversarial fault cases and evidence are in the same tracked history.
7. If a safety invariant fails, stop, repair and repeat the batch's entire relevant QA. Skipped required tests are unverified, not green.
8. Record negative experiments, explicit feature deferrals, and unreproduced external storage faults honestly.
9. No edits to Runwire PHP, Runwire-Core or third-party libraries; no merge/tag/publish without separate maintainer instruction.

### S0 — PIE feasibility and persistent PHP host ownership

**Proposed files:** composer.json, Cargo.toml, Cargo.lock, config.m4 or verified equivalent, rust/php-extension/src/lib.rs, host/*, tests/php/lifecycle_*, docs/adr/0001-host-and-zend-ownership.md.

- [ ] **S0.01** Establish valid PIE php-ext package/build toolchain and a minimal Rust Zend API registration on PHP 8.4 and 8.5; log exact NTS/ZTS and extension ABI for every tested host.
- [ ] **S0.02** Prove a PHP callable accepts canonical cell ID/payload, marshals to Rust and returns typed status/errors without retaining borrowed PHP data after request teardown.
- [ ] **S0.03** Prove safe PHP class activation and method invocation in a persistent host/worker; never invoke the class from an arbitrary Rust Tokio/background thread.
- [ ] **S0.04** Evaluate embedded long-running PHP CLI worker vs managed child process vs companion native service, with process lifecycle, installation, shutdown and operational risks.
- [ ] **S0.05** Test multiple sequential PHP calls, cancellation/exception during handler, request close and PHP module shutdown; no stale zval or callback after owning context ends.
- [ ] **S0.06** Test clean recovery from PHP worker kill -9; distinguish RAM-only state from separately persisted test fixtures and do not claim no data loss yet.
- [ ] **S0.07** Demonstrate native bounded command/completion channel and correct handling of backpressure when no PHP executor is available.
- [ ] **S0.08** Verify fork/process startup ordering for Rust reactor, native mutexes and PHP interpreter; rebuild resources after fork rather than inheriting invalid thread-pool state.
- [ ] **S0.09** Choose provisional isolation boundary for cells from different PHP classes/applications; explicitly exclude hostile multi-tenant safety from MVP.
- [ ] **S0.10** Compare one active cell/worker and several independent cells under PHP user-method load, measuring native crossing overhead, CPU, process RSS and completion fairness.
- [ ] **S0.11** Create initial CI for Rust format/Clippy/tests, clean PIE build/install/uninstall, PHP smoke and repeated startup/shutdown; include sanitizers where feasible.
- [ ] **S0.12** Approve or reject persistent host architecture in a written ADR with operational install/deploy mechanism, failure cases, and a stop/go decision for S1.

**Gate:** a genuinely loaded PIE extension plus a persistent PHP host proof that is safe across requests and worker crashes. No dependency on other Runwire products.

### S1 — cell identities, local actor scheduling and mailbox safety

**Proposed files:** rust/cell-logic/src/identity.rs, mailbox.rs, rust/cell-runtime/src/registry.rs, scheduler.rs, php/stubs/Cells.php, tests/simulation/mailbox_*.

- [ ] **S1.01** Define stable cell ID encoding and canonicalization, type registration, reserved namespace rules and validation for untrusted identifiers.
- [ ] **S1.02** Implement local activation registry and lazily instantiate PHP class/object only when the owning PHP execution context is ready.
- [ ] **S1.03** Define bounded FIFO mailbox semantics, configurable capacity, overflow/reject policy, sender timeouts and deterministic terminal statuses.
- [ ] **S1.04** Enforce one active mutation handler per cell while allowing independent cells on separate PHP executors to progress concurrently.
- [ ] **S1.05** Implement fairness among busy cells so one hot actor cannot starve idle or low-traffic actors.
- [ ] **S1.06** Handle reentrant self-send/self-call explicitly; prohibit unbounded recursion/deadlock or document supported deferred send behavior.
- [ ] **S1.07** Define handler return/throw semantics and cleanup after PHP exception, timeout and actor restart; avoid permanently poisoned mailbox.
- [ ] **S1.08** Define accepted/scheduled/completed/failed/cancelled states, correlation IDs and caller-side deadline behavior without pretending messages are durable.
- [ ] **S1.09** Test full/empty/closed mailbox, simultaneous senders, cancellation races, duplicate IDs, overflow and 1000+ interleaved identities with deterministic scheduling.
- [ ] **S1.10** Benchmark warm in-memory call and contended hot-cell throughput against many independent cells; report queue wait vs PHP method execution cost separately.

**Gate:** No simultaneous mutation on one cell, bounded mailboxes and correct isolation/fairness. **No persistence guarantee yet.**

### S2 — transaction semantics, persistence and repeatable recovery

**Proposed files:** rust/cell-storage/src/{lib,codec,transaction,sqlite,external}.rs or selected backend, rust/cell-logic/src/operation.rs, docs/adr/storage-durability.md, tests/chaos/storage_*.

- [ ] **S2.01** Define storage-mode matrix: volatile local, durable local, and future replicated; specify actual acknowledged commit guarantees, not marketing terms.
- [ ] **S2.02** Compare embedded SQLite and transactional external store under PHP host process model, lock contention, fsync, recovery, state sizes and a future enforced-fencing requirement.
- [ ] **S2.03** Select initial durability backend with explicit transaction implementation, schema, versioning and secure file/credential handling.
- [ ] **S2.04** Create versioned, deterministic cell state codec with integrity/checksum and explicit rejection of PHP resources, closures and nonportable live references.
- [ ] **S2.05** Define operation phases and atomic state transition: read prior version, execute handler, atomically commit state/result/dedup record as needed, then acknowledge.
- [ ] **S2.06** Define durable operation IDs, duplicate detection, retention window and result replay after commit followed by lost RPC response; do not promise perpetual global exactly-once.
- [ ] **S2.07** Implement storage admission and backpressure under slow disk, busy transactions and unavailable database; callers receive bounded failure/timeouts.
- [ ] **S2.08** Define local crash/restart/replay procedure, transactional rollback and snapshot validation; noncommitted state may never be reported committed.
- [ ] **S2.09** Design explicit effect/outbox hook for nontransactional external API calls, avoiding claims that external effects and local state form one atomic transaction.
- [ ] **S2.10** Add kill -9 before commit, after commit before ACK, corrupted snapshot, lost ACK, duplicate replay and disk-full simulated faults.
- [ ] **S2.11** Document read isolation and consistency for concurrent read-only operations versus mutations, including whether reads are serialized or versioned.
- [ ] **S2.12** Benchmark durable single-cell write, many-cell writes, state size growth and fsync cost with p50/p95/p99, RPS and disk write amplification.

**Gate:** Recovery can distinguish committed from uncommitted operations under faults; durable results are acknowledged only after selected backend commit semantics actually hold.

### S3 — activation, eviction, scheduling and footprint

**Proposed files:** rust/cell-runtime/src/{activation,eviction,quota,timers}.rs, rust/cell-storage/src/snapshot.rs, tests/integration/activation_*.

- [ ] **S3.01** Specify cell states absent, activating, ready, busy, quiescing, evicted and failed, with legal transitions and generation IDs.
- [ ] **S3.02** Implement on-demand activation with bounded concurrent activations and explicit cold-start error/timeout policy.
- [ ] **S3.03** Load only required metadata and durable state; never retain one live PHP object or Fiber for every known cell identity.
- [ ] **S3.04** Implement idle eviction after safe completion/transaction boundary; guarantee no accepted message silently dropped.
- [ ] **S3.05** Define activation failure, corrupt state, schema mismatch and retry/backoff; repeated bad cells must not exhaust executors.
- [ ] **S3.06** Implement timers/alarms with explicit volatile vs durable scheduling modes and restart semantics.
- [ ] **S3.07** Track per-cell mailbox/state/working-set bytes, global resident cell count, PHP worker RSS and backend storage footprint.
- [ ] **S3.08** Implement bounded hot-cell, activation and concurrent-worker quotas with documented rejection and priority/fairness policy.
- [ ] **S3.09** Test eviction versus enqueue/commit race, timer fire during restart, reactivation ordering, rapid churn and 100k *known* identities with few residents.
- [ ] **S3.10** Benchmark warm calls, cold activation, hibernated wake, resident memory per active cell and metadata memory per known identity separately.

**Gate:** Resource footprint scales with active work within documented bounds; activation/recovery never violates committed state.

### S4 — local production MVP, auth and operational safety

**Proposed files:** rust/cell-protocol/src/local.rs, rust/php-extension/src/api.rs, host/*, docs/production-local.md, tests/integration/local_mvp_*.

- [ ] **S4.01** Freeze local MVP PHP API only after proven Zend object/method lifecycle and durable operation semantics; generate PHP stubs/reflection tests.
- [ ] **S4.02** Define explicitly allowed cell class registration and callable operation allowlists; no public arbitrary PHP method invocation via remote string.
- [ ] **S4.03** Authenticate/authorize local IPC boundary if present; enforce cell type, identity, operation, payload and caller permissions.
- [ ] **S4.04** Validate body/state/message sizes, mailbox depth, per-cell concurrency, worker process limits and storage budget.
- [ ] **S4.05** Implement reproducible host startup, readiness, graceful stop, config error reporting and safe privilege/permission defaults.
- [ ] **S4.06** Provide bounded diagnostics for activated/evicted cells, mailbox depth, retries, duplicates, durable commits, failed activations and PHP worker crashes.
- [ ] **S4.07** Test multiple independent cell classes and multiple application namespaces without sharing mutable PHP request/tenant state accidentally.
- [ ] **S4.08** Run local soak under mixed hot/cold cells, repeated PHP worker crashes, disk hiccups, storage quota pressure and caller timeouts.
- [ ] **S4.09** Complete PIE packaging for the supported PHP 8.4/8.5 local target, plus any independently required host service installation documentation.
- [ ] **S4.10** Publish local MVP acceptance matrix, semantics, unsupported tenancy/clustering claims, resource limits and raw benchmark evidence.

**Gate:** A usable, independently installable single-host stateful PHP actor engine; no multi-node guarantees asserted.

### S5 — local multi-process placement, IPC and worker recovery

**Proposed files:** rust/cell-runtime/src/placement.rs, ipc.rs, worker.rs, rust/cell-logic/src/generation.rs, tests/chaos/worker_*.

- [ ] **S5.01** Define multiple PHP worker ownership and selection policy for independently addressable cells within one machine.
- [ ] **S5.02** Implement a process-local directory associating cell key with worker generation, live address, last successful commit and activation status.
- [ ] **S5.03** Define versioned, authenticated local RPC/IPC envelopes, bounded payload sizes and schema errors.
- [ ] **S5.04** Implement send/reply timeout, caller disconnect, duplicate request ID and pending-message error/failover behavior.
- [ ] **S5.05** Ensure failed worker cannot keep mutating state after replacement by generation checks plus storage-level transaction authority where needed.
- [ ] **S5.06** Implement worker crash restart/backoff, surviving mailboxes semantics, rescheduling policy and recovery from last acknowledged durable state.
- [ ] **S5.07** Prove in-flight PHP object destructors/cancellation do not run after VM teardown and native RPC responses do not dereference stale handles.
- [ ] **S5.08** Test repeated kill -9 during write, worker pause/resume, version mismatch and local IPC packet loss/disconnection.
- [ ] **S5.09** Load-test multi-worker fairness and hot-cell routing under worker recycling; quantify IPC overhead separately from PHP method and storage time.
- [ ] **S5.10** Document local failure domain, active writer and worker generation guarantees without claiming cluster-wide exclusivity.

**Gate:** Safe local placement/restart, recovered correct durable state and no duplicate local active writer under tested faults.

### S6 — independent cluster transport, routing and membership

**Proposed files:** rust/cell-protocol/src/{wire,auth,errors}.rs, rust/cell-cluster/src/{membership,routing,transport}.rs, tests/integration/cluster_*.

- [ ] **S6.01** Specify standalone cluster topology, discovery/membership assumptions and actual external service requirements. Do not require Runwire-Core.
- [ ] **S6.02** Define versioned RPC envelope: cell ID, operation ID, caller deadline, caller identity, routing epoch, failure classification and payload limit.
- [ ] **S6.03** Enforce production node-to-node authentication, encryption, replay protection and credential rotation; fail closed without credentials.
- [ ] **S6.04** Implement read-only membership observations and routing cache with bounded staleness; routing cache alone must never authorize mutating writes.
- [ ] **S6.05** Define client redirection/forwarding semantics for wrong-node request, hop limit, stale route and cell moved during request.
- [ ] **S6.06** Implement backpressure and retry budget for peer connections, bounded in-flight RPCs, circuit-breaker-like overload and jittered reconnects.
- [ ] **S6.07** Separate routing unavailability from storage/ownership failure; no blind retry of non-idempotent requests without operation IDs.
- [ ] **S6.08** Test incompatible protocol versions, oversized messages, unauthorized operations, expired certificates, peer reconnect storms and malformed RPCs.
- [ ] **S6.09** Benchmark one-hop vs remote-hop warm reads/writes with identical durability, network RTT and PHP host allocation.
- [ ] **S6.10** Record the cluster protocol support matrix, credentials/configuration and precise undecided consistency questions deferred to S7.

**Gate:** Secure and bounded multi-node transport/routing exists; **no multi-owner write mode is allowed before fencing is proven**.

### S7 — lease authority, exclusive writer and enforceable fencing

**Proposed files:** rust/cell-cluster/src/{authority,epoch,lease,takeover}.rs, rust/cell-storage/src/fenced_commit.rs, tests/simulation/ownership_*.

- [ ] **S7.01** Choose the authoritative source of ownership and epoch generation (conditional writes, transactional storage authority or consensus service); justify linearization point.
- [ ] **S7.02** Define owner state machine unowned → acquiring → active → suspect → fenced → recovering → active/new owner; model invalid transitions.
- [ ] **S7.03** Guarantee monotonic fencing token generation and durable storage validation on **every relevant commit**, not merely on lease acquisition.
- [ ] **S7.04** Prove stale owner cannot commit after the authority has advanced to a successor generation, even if paused worker resumes.
- [ ] **S7.05** Distinguish lease clocks, grace period, storage stalls and clock skew. Never use local clock or expiry alone to authorize stale writes.
- [ ] **S7.06** Make cell mutating execution fail closed when ownership/authority cannot be established; define read-only behavior separately.
- [ ] **S7.07** Implement idempotent takeover with stale message rejection, generation-tagged routing and recovery of previous owner in-flight operations.
- [ ] **S7.08** Test simultaneous acquisition by 2+ nodes, crashed leader, delayed renewal, lost lease response, packet partition and old owner resume.
- [ ] **S7.09** Inject long conditional-store pauses, short TTL, slow disk and full network partition; verify at most one **committable** writer per cell.
- [ ] **S7.10** Build deterministic state-machine simulation with invariant checks; consider formal TLA+ or equivalent model for owners/epochs/takeover.
- [ ] **S7.11** Record availability consequence: partitions may prevent writes to preserve safety. Never automatically trade split-brain avoidance for optimistic uptime.
- [ ] **S7.12** Publish proof/evidence of fencing invariant, recovery trace and known backend assumptions before enabling distributed writes by default.

**Gate:** Stale owner writes rejected at authoritative store under deterministic and real fault injection; otherwise distributed mutating service is blocked.

### S8 — replication, durability modes, recovery and RPO/RTO

**Proposed files:** rust/cell-storage/src/{replication,snapshot,replay,recovery}.rs, rust/cell-cluster/src/recovery.rs, tests/chaos/durable_*.

- [ ] **S8.01** Define and separately name durability modes: synchronous transactional external store, local-only, peer-replicated or object-store backed **only if actually implemented**.
- [ ] **S8.02** Specify write acknowledgment rule per mode (local WAL/fsync, remote quorum/transaction, bucket receipt) and what happens on insufficient acknowledgments.
- [ ] **S8.03** Evaluate replication strategy against costs, snapshot/log integration, recovery integrity and operational complexity; do not assume celld LTX design is automatically transferable.
- [ ] **S8.04** Implement state versioning, snapshot/log reconciliation and checksum chain if using logs; detect divergence instead of silently choosing a winner.
- [ ] **S8.05** Implement recovery with epoch fencing: new owner must restore committed state before accepting new mutating calls.
- [ ] **S8.06** Define replay and deduplication of committed operations after ACK lost, interrupted replication and retry from old routing epoch.
- [ ] **S8.07** Exercise storage outage, delayed object-store responses, partial replication, snapshot corruption, disk full, network loss and process kill at each commit phase.
- [ ] **S8.08** Measure acknowledged operation loss (RPO), time until safe serving (RTO), data checksum and last-committed ID across real multi-node node-failure trials.
- [ ] **S8.09** Test cold restore and large cell state hydration with sizes from small counters to multi-GB data where practical; bound CPU, memory and progress reporting.
- [ ] **S8.10** Prevent a slow/corrupt cell from starving recovery of unrelated cells; enforce per-node recovery concurrency and byte limits.
- [ ] **S8.11** Document exact guaranteed/unsupported failure domains: whole cluster outage, storage provider unavailable, disk loss, region loss and multiple simultaneous failures.
- [ ] **S8.12** Reject unqualified "RPO=0", "exactly once" or "always available" statements; publish claims only after tests for explicitly named topology/durability mode.

**Gate:** Every offered durability mode has an executable recovery/failure contract with measured RPO/RTO and no stale-owner commits.

### S9 — operations, rebalancing, upgrades and state evolution

**Proposed files:** rust/cell-cluster/src/{rebalance,drain,upgrade}.rs, rust/cell-storage/src/migration.rs, host/config/*, docs/operations.md.

- [ ] **S9.01** Define placement policy and capacity metrics: PHP executors, active cell count, memory/RSS, state size, queue pressure and storage latency.
- [ ] **S9.02** Implement safe planned rebalance: quiesce old owner, release/fence, restore new owner, update route and resume without duplicate write acceptance.
- [ ] **S9.03** Implement node drain/rolling restart with bounded grace periods, explicit incomplete RPC outcomes and durable queues where contracted.
- [ ] **S9.04** Specify PHP cell class/schema versioning and data migration with checkpoint/rollback; version mismatches fail deterministically.
- [ ] **S9.05** Define backward/forward wire compatibility, old PHP deployment vs new native host and mixed-version cluster upgrade procedure.
- [ ] **S9.06** Implement backup/export and recovery testing with integrity checks, keys/secrets and clearly defined consistency point.
- [ ] **S9.07** Add operational observability: owner epoch, placement, warm/cold rates, queue wait, pending commit, auth failures, failovers and recovery backlog.
- [ ] **S9.08** Validate administrative API least privilege and controls to quarantine, inspect and safely recover corrupted or hot cells.
- [ ] **S9.09** Soak-test incremental scaling, node join/leave, rolling upgrades, repeated rebalancing and large numbers of idle identities.
- [ ] **S9.10** Produce operator runbook with install/config, deployment, network/storage prerequisites, cert/key rotation, incident response and rollback.

**Gate:** Operational changes cannot violate single-writer/durability invariants; backup/restore and safe rolling upgrades are demonstrated.

### S10 — independent release certification and packaging

**Proposed files:** .github/workflows/*, composer.json, config.m4 or equivalent, Cargo.lock, README.md, docs/install.md, docs/benchmarks.md, docs/release.md.

- [ ] **S10.01** Verify independent package identity, namespace, PHP module name, PIE php-ext manifest and explicit optional companion-service distribution.
- [ ] **S10.02** Run clean PIE source install and extension load/unload for declared Linux/PHP 8.4/8.5/NTS/ZTS targets, with accurate unsupported-platform errors.
- [ ] **S10.03** Verify companion host provisioning is separate, auditable and unprivileged by default; an installed extension must not automatically expose a network service.
- [ ] **S10.04** Run Rustfmt, Clippy, dependency/security audit, Rust unit/property/model checks, fuzz smoke and real PHP lifecycle tests on exact candidate SHA.
- [ ] **S10.05** Run local actor API conformance, all durable storage modes, multiworker restart and full multi-node authenticated transport/fencing/recovery suites **only for actually advertised features**.
- [ ] **S10.06** Exercise adversarial fault matrix: paused stale writer, partitions, conditional-store stall, ACK loss, worker kill -9, corrupt snapshot, disk full, slow hot cell and dual takeover.
- [ ] **S10.07** Publish complete benchmark evidence: warm/cold p50/p95/p99, committed ops/sec, memory per resident and known cell, CPU, disk/network, failover RTO and achieved RPO.
- [ ] **S10.08** If comparing with celld, match durability/replication mode, workload, client, network, node topology and storage behavior; report limitations, not winner-only numbers.
- [ ] **S10.09** Document class API, host install, state and message schema, acknowledgement model, call/cancel semantics, storage setup, TLS/auth, placement and scaling.
- [ ] **S10.10** Publish limitations and trust boundaries: no arbitrary hostile PHP sandbox, no magic exactly-once external effect, no guarantee under unsupported partition/storage mode.
- [ ] **S10.11** Review skipped/failing CI gates and release blockers; distinguish local-only MVP readiness from distributed readiness. Do not mark clustering done just because local actor API passes.
- [ ] **S10.12** Prepare release handoff with candidate SHA, artifacts, changelog, license/dependency notices, supported matrix and human go/no-go. Leave branch unmerged, untagged and unpublished until maintainer authorizes.

**Gate:** Support claims match independently verified production topology, durability and PHP ABI; no high-severity unresolved safety flaws.

### Stage-by-stage deliverable acceptance, separated from the final distributed goal

| Gate | Earliest milestone | Allowed claims | Not yet permitted claims |
|---|---|---|---|
| Host proof | S0 | PIE loads and managed PHP context can invoke cells safely | durable actors, distributed or production |
| Local volatile actors | S1 | serialized in-memory per-cell mailbox with bounded admission | crash-safe state |
| Durable local actors | S2–S4 | tested transactional state recovery on a specified single host | multi-node owner guarantees |
| Local multiworker | S5 | process replacement with tested local recovery | partition-safe distributed writes |
| Cluster RPC | S6 | remote address/routing, authenticated transport | split-brain safety without S7 |
| Fenced cluster | S7 | tested exclusive committable ownership under named failures | RPO/RTO promise without S8 |
| Recoverable distributed cells | S8–S10 | storage-mode-specific measured recovery and durability | universal zero loss or exactly once |

### QA and performance acceptance matrix

| Axis | Required scenarios | Primary indicators |
|---|---|---|
| Zend/PIE | extension install/load, repeated requests, worker recycle, NTS/ZTS, exception/bailout | no UAF, no invalid PHP callback |
| Identity/mailbox | collisions, self-call, overflow, fairness, independent hot cells | serialization, correct replies, stable queue |
| Persistence | crash before/after commit, ACK loss, duplicates, corrupt state | checksum, commit ID, durable ACK correctness |
| Lifecycle | activation, hibernation, rapid churn, alarms, timers | cold/wake p99, resident bytes, no loss |
| Multinode transport | TLS/auth, redirection, stale cache, wrong epoch, lost peer | correct rejection, no retry storm |
| Ownership | two simultaneous owners, lease pause, clock drift, partition | storage-enforced fenced writes |
| Recovery | node/data store crash, partial replication, large snapshot | tested RPO/RTO and data integrity |
| Operations | scale, rebalance, rolling version change, backups | no duplicate writes and bounded drains |
| Performance | warm/cold/durable/hot-cell/many-cell curves | p50/p95/p99, successful committed ops/sec, CPU/RSS |

### Evidence manifest and progress synchronization

For each Sx.yy task store: status (NOT STARTED, IN PROGRESS, BLOCKED, DONE or REJECTED WITH EVIDENCE), dated decision and exact source SHA, touched paths, Rust toolchain, PHP ABI, storage mode/topology, reproducible tests, failure injection, CI artifact links, full workload/results, known unsupported cases and review verdict. The **S0–S10 summary table must stay synchronized** with these numbered tasks and actual gate outcomes in each implementation commit.

**This detailed plan creates no implementation, daemon, extension package, cluster or certified durability claim. Every task remains open until evidence is collected.**


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
