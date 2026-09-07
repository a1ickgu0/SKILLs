# Agent Prompt Templates

Five templates covering dimensions A–E. Every template shares the
same skeleton; only the dimension-specific task block differs.

## Shared skeleton (all templates)

```
You are analyzing <DIMENSION> for PERFORMANCE in <COMPONENT>. This is READ-ONLY
research — do NOT modify any code. Write findings to {OUTPUT_FILE} (English, tables
preferred, <400 lines).

Working dir: {WORKDIR}

SCOPE RESTRICTION (mandatory): You may ONLY read files under these directories:
1. {ALLOWED_DIR_1}
2. {ALLOWED_DIR_2}          # e.g. SDK/dependency headers
Do NOT read files anywhere else. If a definition lives outside the allowed
directories, note "[external, not read]" and move on.

{ENCODING_WARNING}

CONTEXT already established (do not re-derive):
{CHECKPOINT_FACTS}          # bullet list with file:line, copied from checkpoint

DIMENSION TASKS (cite file:line for every claim; mark inferred claims [inferred]):
{TASK_BLOCK}                # from the template below

OUTPUT FORMAT:
- The sections named in the task block, tables preferred
- Bottleneck list: severity high/med/low + evidence per item (emit severity tags
in the report's language per reporting-standards §3)
Report when done in under 150 words.
```

Placeholders: `{WORKDIR}` session cwd; `{ALLOWED_DIR_*}` from the user-approved scope;
`{CHECKPOINT_FACTS}` dimension-relevant checkpoint slice; `{OUTPUT_FILE}` e.g.
`docs/analysis/agent_A_timers.md`; `{ENCODING_WARNING}` emit the warning line
ONLY when the Phase 0 encoding probe (`file -bi` on a few sources) detects a
non-UTF-8 charset; use this exact wording when emitted: "ENCODING WARNING: files
may be <charset> encoded — ALWAYS use `grep -a` (text-mode) or searches will
silently return nothing." Otherwise omit the line entirely.

Abstraction vocabulary: use "entry" for the component's primary stored object
(route/prefix/flow/session/call), "peer/downstream" for replication targets, "control
plane" for the decision path, "data path" for the processing path.

---

## T-A — Timers & periodic tasks

```
1. Timer infrastructure: find the scheduler integration (thread_add_timer or
   equivalent). Map ALL periodic timers (keepalive/heartbeat, expiry/aging,
   scan/refresh, retry/backoff) into an inventory table: name, owning thread,
   interval, trigger, file:line.
2. Per-expiry cost for the 3 hottest timers: what runs at expiry, per-registration
   cost, aggregate cost = per-registration × N registrations (compute at the
   design-point N, e.g. 2K peers/sessions/flows).
3. Scan-type timers (periodic full-table walks): what does each scan iterate,
   what is the cost at 1M entries, is the walk time-sliced (budget/quota) or blocking?
4. Timer-storm risks: per-registration timers (N×M bookkeeping) vs coalesced lists;
   sorted-list insert cost O(N)? double registration patterns?
5. Backoff/aggregation timers (rate-limit, min-interval, debounce): re-arm behavior
   under sustained load — event-per-second arithmetic.
6. Watermark re-arm loops: timers reused for backpressure re-checks — frequency at
   design point.
OUTPUT: Section 1 inventory table; Section 2 top-3 expiry costs; Section 3 scan
walks; Section 4 risk list (severity + evidence).
```

## T-B — Data structures & algorithm complexity

```
1. Core container inventory: primary storage (tree/hash/list/bitmap) for the main
   entry table; struct-per-entry byte estimate (fields summed, [inferred] per-struct
   byte math); lazy/optional layers (extra/extension structs) and when allocated.
2. Operation complexity table: insert / lookup / delete / iterate, algorithm,
   complexity, file:line.
3. Index asymmetry: per-entry lookup indexes (hash vs linear) — check both ingress
   and egress/replica sides; a missing index on one side is a classic finding.
4. Per-lookup-key resolution: direct-index array vs list walk for ID→object.
5. FIFO/queue structures for deferred work: O(1) enqueue? dedup flags? priority tiers?
6. O(N²)/O(N×M) risks: peer/replica walks inside entry loops, pairwise comparisons
   (e.g. deterministic-MED-style passes), attribute/policy comparisons in selection.
7. Attribute/policy object size + interning (shared hash): bucket count, comparison
   strategy (memcmp vs field-by-field), load-factor risk.
OUTPUT: Section 1 struct inventory; Section 2 complexity table; Section 3 O(N×M)
risks; Section 4 memory per 1M entries [inferred, show arithmetic]; Section 5
bottleneck list.
```

## T-C — Memory & IPC overhead

```
1. Allocator & pool inventory: allocator entry chain (XMACRO → platform malloc);
   custom pools (types pooled, chunk sizes, retention caps, reaper/drain rates);
   which hot-path structs are NOT pooled.
2. Per-entry allocation count: trace create/update/withdraw lifecycle — how many
   heap allocations per entry learn; what multiplies it (per-peer adjacency,
   soft-reconfig, backup modes).
3. Attribute/policy sharing: interning hash sizes; per-entry persistent memory =
   entry struct + share pointer if interned.
4. IPC copy chain: trace one message from decode to egress thread/socket; count
   buffer copies; inline arrays in IPC structs that force whole-payload copies;
   flush-timer latency.
5. Backup/HA multiplier: sync buffers, unack lists, standby dumps — memory
   multiplier per entry or per packet; find watermark thresholds and their
   arithmetic (often documented in comments).
6. Leak-pattern scan: withdraw free-chain completeness; grow-only buffers.
OUTPUT: Section 1 allocator inventory; Section 2 allocation count trace; Section 3
IPC copy diagram; Section 4 HA multiplier; Section 5 bottleneck list. Mark any
sub-area not covered as [not analyzed].
```

## T-D — Concurrency, locks & event loop

```
1. Thread & channel inventory: every real thread (pthread_create) and pseudo-thread
   domain; what each owns; every cross-thread channel (queue, shm, pipe).
2. Event-loop walkthrough: read the egress/IO thread main loop; what it polls
   (epoll/level vs edge), per-iteration cost, read/write batching per iteration.
   Pseudo-code it.
3. Critical-section audit: every lock site — work done under lock (lines, syscalls
   like socket writes inside lock = red flag), contention risk at design point.
4. Cross-thread signal audit: every shared volatile/atomic — writer, reader, race
   assessment; note absence of atomics/barriers.
5. Backpressure & watermarks: high/low thresholds, who blocks/pauses, deadlock
   possibility, thundering-herd on recovery.
6. Show/control paths crossing threads: message granularity, chattiness, priority
   (can control traffic preempt data?).
OUTPUT: Section 1 inventory; Section 2 loop walkthrough; Section 3 lock audit;
Section 4 signal audit; Section 5 backpressure; Section 6 bottleneck list.
```

## T-E — Business critical path

```
Adapt to the component: BGP→route convergence, firewall→rule commit, LB→session
setup, PBX→call setup. Confirm the path with the user if unclear.

1. End-to-end pipeline diagram (text) with stage file:line: ingress decode →
   validation → decision/selection (bestpath/state compute) → replication/build →
   egress.
2. Per-stage cost: top-5 hottest stages (policy/route-map evaluation, hash-key
   recomputation, comparisons).
3. Deferral & batching: is decision immediate or deferred (FIFO)? batch sizes,
   scheduler quanta, budgets, priority tiers, fairness (global vs per-peer queue
   = head-of-line risk).
4. Replication economics: shared encode vs per-peer rebuild; what breaks grouping
   (per-peer policy evaluation); per-peer copy counts.
5. Feature multipliers on the path: GR/NSR/high-availability stale handling,
   refresh/replay, dampening bookkeeping even when disabled.
6. Capacity estimate [inferred, show arithmetic]: full-population scenario (1M
   entries), CPU time and wall-clock, what dominates.
OUTPUT: Section 1 pipeline; Section 2 stage costs; Section 3 batching deep-dive;
Section 4 replication; Section 5 capacity estimate; Section 6 bottleneck list.
```

Ad-hoc deep-dives (Phase 2, letters F/G/…) have no canned template: decompose the
user's concern into a numbered task block following the T-A…T-E conventions —
sections, tables, `[inferred]` marking, severity+evidence output contract. For
memory-fragmentation-style concerns, the field-tested decomposition is: byte-exact
sizing → allocator-wrapper assumptions → internal/external fragmentation →
second-layer cache interaction → aggregate scenarios → ranked recommendations.

## Pre-dispatch dry-run checklist

Before dispatching any instantiated agent, the orchestrator runs this mechanical
self-check. A missing scope block is the direct cause of most scope-audit
failures, so check #1 first.

1. The instantiated prompt contains all seven skeleton elements: role +
   output-file; SCOPE RESTRICTION block with verbatim allowed dirs;
   `{ENCODING_WARNING}` or its deliberate omission; `{CHECKPOINT_FACTS}` slice;
   task block with citation requirement; OUTPUT FORMAT contract; 150-word reply
   constraint.
2. Every allowed directory in the scope block exists on disk.
3. The `{OUTPUT_FILE}` path is inside the analysis output dir.
4. Checkpoint facts cited in the prompt carry file:line.

