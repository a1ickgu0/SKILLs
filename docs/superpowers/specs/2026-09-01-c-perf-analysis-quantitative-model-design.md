# c-perf-analysis Quantitative Analysis Layer — Design Spec

## Status

- Design direction approved by the user on 2026-09-01.
- This document specifies the upgrade; it does not modify the skill yet.
- Implementation must follow RED-GREEN-REFACTOR behavioral testing for skills.

## Overview

Upgrade `c-perf-analysis` from a strong static-audit orchestration skill into a
reproducible quantitative-analysis skill. Preserve its serial orientation,
parallel A–E dimensions, evidence discipline, and cross-agent reconciliation.
Add one shared quantitative layer that turns source facts and software-engineering
knowledge into scenario-bound formulas, ranges, confidence labels, and verification
hooks.

The core contract is:

```text
source/build evidence
  → semantic fact
  → scenario variable
  → per-operation cost or work-unit vector
  → multiplicity/fan-out/lifetime model
  → resource metric and bounds
  → confidence and verification hook
```

Static analysis must not manufacture CPU time, utilization, RSS, stable queue wait,
or p99 latency when the coefficients needed for those quantities have not been
calibrated.

## Baseline Findings

The existing skill already provides valuable foundations:

- A–E coverage for timers, data structures, memory/IPC, concurrency, and the
  business critical path.
- `file:line` citations, `[inferred]`, `[external, not read]`, and `[not analyzed]`.
- Scenario-style capacity arithmetic, metric reconciliation, and dynamic
  verification suggestions.
- A serial checkpoint that feeds parallel agents and avoids duplicate exploration.

The RED baseline exposed gaps that are not enforced by the current instructions:

1. There is no explicit gate that rejects an exact CPU percentage, RSS, stable
   wait, or p99 when required inputs are missing.
2. ABI layout, flexible arrays, allocator rounding, allocator resident memory,
   and process RSS lack a shared layered model.
3. CPU percentage lacks a mandatory denominator; CPU time, wall-clock time, and
   input-arrival time can be conflated.
4. Hash load factor, hit/miss probe counts, queue stability, batch-fill delay,
   lock demand, and latency DAG composition lack shared formulas and preconditions.
5. Big-O findings do not consistently recover call frequency, nested-loop bounds,
   peer/subgroup fan-out, retry, copy, and object-lifetime multipliers.
6. Evidence kind and confidence are collapsed into the single `[inferred]` tag.
7. Phase 3 reconciles conflicting values but does not require arithmetic closure,
   unit closure, duplicate-component detection, or scenario consistency.

The current demo proves the last point. `demo_2_metric_reconciliation.md` states a
`656 B` total for terms listed as `248 + 128 + 64 + 40 + 32`, which sum to `512 B`.
The report also describes allocator slack as included in the `248 B` floor in one
place and excluded in another. `demo_1` and `demo_2` use different `adj_out` sizes
(`80 B` and `64 B`) without converting the difference into an explicit range.

## Goals

1. Make every important numeric claim reproducible from named inputs and a formula.
2. Make scenario, build, ABI, allocator, workload mode, units, and statistical
   semantics explicit.
3. Apply C, data-structure, architecture-pattern, and experienced-coding knowledge
   through reusable rule cards rather than scattered reminders.
4. Express static-only conclusions honestly as counts, work-unit vectors, symbolic
   expressions, ranges, or conditional bounds.
5. Permit time/utilization/throughput/percentile claims only when the required
   coefficients are dynamically calibrated or explicitly modeled as experience-based
   assumptions.
6. Detect arithmetic, dimensional, scope, fan-out, double-counting, and scenario
   errors before consolidation.
7. Preserve progressive disclosure: `SKILL.md` remains the workflow entry point;
   heavy formulas and rule cards live in directly linked references.

## Non-goals

- Replacing `perf`, heap profilers, traces, load tests, or production telemetry.
- Automatically proving exact hardware behavior from C source.
- Requiring a particular allocator, compiler, architecture, or build system.
- Turning the skill into a general C correctness or security audit.
- Modifying the analyzed component; the skill remains analysis-only.
- Making microarchitectural claims before a hotspot has been established.

## Information Architecture

Keep all runtime references one level below `SKILL.md`.

```text
c-perf-analysis/
├── SKILL.md
├── references/
│   ├── quantitative-analysis-model.md    # NEW: scenarios, units, formulas,
│   │                                      # metric records, evidence, validation
│   ├── software-performance-rules.md     # NEW: C/DS/pattern/experience rule cards
│   ├── agent-prompt-templates.md         # MODIFIED: inject shared contracts
│   ├── reporting-standards.md            # MODIFIED: render validated metrics
│   ├── orchestration-checklist.md         # MODIFIED: Phase-0/3 gates
│   └── glossary.md                        # MODIFIED only for new stable terms/tags
├── scripts/
│   └── validate_metrics.py               # NEW: deterministic mechanical checks
└── demo-case/
    ├── evaluations/
    │   └── quantitative-scenarios.md      # NEW: RED/GREEN behavior cases
    └── reports/
        ├── demo_1_pipeline_teardown.md    # MODIFIED
        └── demo_2_metric_reconciliation.md# MODIFIED
```

`quantitative-analysis-model.md` and `software-performance-rules.md` must include a
contents table because they will exceed 100 lines. `SKILL.md` links both directly and
states exactly when each must be read.

## Shared Quantitative Contract

### Scenario ledger

Phase 0 creates at least one named `scenario_id`. A scenario records:

- source revision and effective build configuration;
- target ABI, pointer width, compiler/optimization mode, and allocator;
- workload mode: `steady`, `burst`, or `recovery`;
- observation window and scale variables;
- input/event rates, entry/peer/subgroup counts, message/payload distributions;
- enabled, disabled, and unknown feature flags;
- hardware profile when dynamic coefficients are used.

Scenario parameters are inputs, not source evidence. Values from different scenarios
must not be combined unless Phase 3 constructs and names a reconciliation scenario.

### Semantic units

Counts retain their entity type: `count(entry)`, `count(peer)`, `count(message)`, and
`count(batch)` are not interchangeable dimensionless values. Standard units include:

- bytes as `B`, with `MB` and `MiB` distinguished on display;
- rates such as `count(event)/s` and `B/s`;
- time as `ns`, `us`, `ms`, or `s`;
- typed work units such as `hash_probe`, `compare`, `visit`, `alloc`, `copy_B`,
  `syscall`, `lock_acquire`;
- CPU demand as effective core equivalents only after per-unit calibration.

Mean, peak, p50, p95, p99, and hard upper bound are different metric semantics.

### Evidence and confidence

Evidence kind and confidence are independent fields:

| Kind | Meaning | Permitted result |
|---|---|---|
| `SF` | Static fact from source, config, compiler output, or binary | constants, paths, exact configured layout |
| `SB` | Static bound derived only from static facts and scenario inputs | counts, bytes, work units, conditional bounds |
| `EA` | Experience-based assumption or heuristic coefficient | labeled assumption-model range only |
| `DC` | Dynamic calibration for a named build/scenario/hardware profile | time, cycles, distributions, measured slopes |

Confidence is `C3` high, `C2` medium, `C1` low, or `C0` unsupported. The most
material uncertain input governs the aggregate confidence. Dynamic calibration upgrades
only the measured coefficient; it does not repair an incomplete static path.

### Metric record

Each high-severity quantitative finding and each number used in the executive summary
must have a machine-readable metric record. Each agent writes its own metrics file to
avoid parallel write conflicts.

```text
agent_A_timers.md
agent_A_metrics.json
...
```

A metric record requires:

- `metric_id`, dimension, name, and `scenario_ref`;
- result status (`exact`, `range`, `bound`, `symbolic`, or `unsupported`) and unit;
- formula ID and structured inputs;
- value/range and visible derivation;
- provenance for every non-literal input;
- evidence kind and confidence;
- `includes` and `excludes` lists;
- assumptions and prohibited interpretations;
- verification hook and validation status.

Agents may place lower-severity qualitative findings only in Markdown. Numeric summary
tables may consume only metric records that passed validation.

## Formula Families

The shared reference defines formulas and their preconditions for:

| Family | Static output | Calibrated extension |
|---|---|---|
| CPU | typed work-unit vector and call counts | CPU time, utilization, CPU capacity bound |
| Memory | requested live bytes, object counts, symbolic allocator layers | usable/resident/RSS slopes and fragmentation |
| IPC | messages/event, logical/copied/wire bytes | copy CPU and handoff latency |
| Timers | expiry/s, visits/s, storm count/window | callback CPU and observed loop lag |
| Queue | capacity bytes, arrival count, service work units | utilization, drain time, stable wait |
| Locks | acquisitions/event, critical-section work units, serialized topology | hold/wait distributions and contention |
| E2E latency | stage DAG and configured gating bounds | service/queue distributions and percentiles |

Required guardrails include:

- Big-O cannot be added to time.
- Requested bytes, allocator usable bytes, allocator resident bytes, and process RSS
  are separate layers and must not be summed as siblings.
- `rho < 1` must be established before finite steady-state queue wait is reported.
- Parallel DAG branches use `max`; serial stages sum.
- A periodic flush contributes `[0,T]` statically; `T/2` requires an explicit
  independent/uniform phase assumption.
- CPU time divided by core count is an ideal lower bound, not wall-clock proof.
- If a serial stage is unbounded, the end-to-end static upper bound is unbounded.

## Software Performance Rule Cards

The rule library applies software-development knowledge through a uniform card:

```text
ID / priority / applicable A–E dimensions
recognition signals
required code and build evidence
cost formula or upper/lower bound
common misread
verification hook
```

P0 cards, checked whenever their signal is present:

- effective preprocessing/build configuration;
- ABI layout, padding, flexible arrays, and true allocation stride;
- object lifetime, ownership, live set, error/withdraw free chain;
- allocation count, allocator size classes, pools, and retained memory;
- data-structure real constants, load factor, comparisons, and cache locality;
- hidden nested loops, callbacks, fan-out, retries, and replication multipliers;
- Reactor loop budget and blocking work inside callbacks;
- producer-consumer queue stability and burst backlog;
- pipeline bottleneck, copy chain, and stage ownership;
- state-machine transition amplification;
- batching, quotas, flush timers, and backpressure;
- timer storms, full scans, and re-registration;
- lock-held allocation/logging/syscalls/callbacks;
- error, timeout, reconnect, and retry feedback loops.

P1 cards are signal-triggered: dynamic buffer growth, multiple indexes, object pools,
Flyweight/interning, hot-path logging, temporary zero/copy work, and false sharing.

P2 cards require an established hotspot: branch prediction, vectorization, inlining,
instruction cache, prefetch, NUMA placement, and ISA/code-generation details.

Phase 0 selects relevant rule IDs. Agent prompts receive only that dimension's selected
cards rather than the whole rule library.

## Workflow Changes

### Phase 0

Add build/ABI/allocator discovery, scenario ledger, active feature configuration, and
selected rule IDs to `analysis_checkpoint.md`. Unknowns are preserved as unknowns rather
than replaced by defaults.

### Phase 1

Each A–E prompt receives:

1. its existing scope and evidence block;
2. the relevant scenario slice;
3. selected rule cards;
4. allowed static output types for that dimension;
5. the metric-record contract and metrics output path.

The existing Markdown report remains the human-readable analysis. Structured records
contain the reusable quantitative claims.

### Phase 2

Ad-hoc deep dives reuse the same scenarios and metric contract. A new scenario is
required when a deep dive changes allocator, hardware, build flags, workload mode, or
feature configuration materially.

### Phase 3

Run metric validation before reconciliation. Reconciliation then distinguishes:

- arithmetic errors;
- unit or statistical-semantics errors;
- different scenario/build/feature scopes;
- different included/excluded components;
- legitimate alternative assumptions.

Only the last two normally become a reconciled range. Arithmetic and unit errors must be
fixed, not presented as equally valid opinions.

## Deterministic Validation

`scripts/validate_metrics.py` uses only the Python standard library and validates each
`agent_*_metrics.json` file. It must report exact metric IDs and fields for:

1. schema completeness;
2. arithmetic closure for supported formula IDs;
3. unit closure;
4. scenario/build consistency;
5. range propagation and significant-digit limits;
6. duplicate component IDs and parent/child double counting;
7. fan-out ownership (`per-entry`, `per-peer`, `per-subgroup`);
8. steady/peak/percentile semantic mixing;
9. CPU-time versus wall-clock interpretation;
10. queue stability preconditions;
11. missing verification hooks for `EA` and uncalibrated metrics.

The validator does not evaluate arbitrary expressions. It recognizes a bounded formula
catalog and rejects unknown formula IDs for summary-grade metrics. Qualitative analysis
and exploratory formulas remain in Markdown until promoted into the catalog.

## Reporting Changes

The final report adds:

- scenario/build profile table;
- quantitative profile with metric ID, result, unit, evidence kind, confidence, and
  validation status;
- sensitivity and break-even variables for high-severity findings;
- explicit CPU time versus wall-clock, requested versus resident memory, and
  logical versus copied versus wire bytes;
- assumption ledger tied to metric IDs;
- expected benefit stated as a formula/range, not an unsupported percentage.

Severity remains impact-based, but a High severity does not imply High confidence.

## Behavioral Evaluation

The committed evaluation set covers at least these cases:

1. Flexible-array object with payload percentiles and an unsupported request for exact
   RSS. Expected behavior: preserve layout/allocator/RSS layers and refuse false precision.
2. Fixed-bucket chained hash at high load with an unsupported request for CPU percent.
   Expected behavior: compute probe-count/work-unit bounds and require calibration for time.
3. Reactor holding a mutex across `send()` with missing service distribution and a demand
   to prove no backlog/p99. Expected behavior: state stability conditions and reject proof.
4. Contradictory memory totals modeled after the current demo. Expected behavior: fail
   arithmetic closure instead of reconciling both numbers.
5. Mixed serial/parallel pipeline and flush timer. Expected behavior: construct a DAG,
   avoid summing parallel branches, and separate configured gating from measured p99.

RED preserves current-skill outputs and omissions. GREEN reruns the same cases with the
updated skill. Wording micro-tests use fresh contexts and include a no-new-guidance control.

## Acceptance Criteria

- Every executive-summary number maps to a validated metric record.
- Every metric record has a scenario, formula, typed units, provenance, evidence kind,
  confidence, scope, assumptions, and verification hook.
- Unsupported exact RSS/CPU utilization/throughput/wait/p99 requests result in symbolic
  or conditional output, not invented constants.
- The validator catches the existing `656 B` arithmetic inconsistency.
- A–E agents do not duplicate the full rule library; they receive selected cards.
- Phase 3 distinguishes invalid arithmetic from legitimate scope differences.
- Updated demos pass arithmetic, unit, scenario, and duplicate-component checks.
- Skill frontmatter remains discoverable and describes trigger conditions rather than
  summarizing the workflow.
- `SKILL.md` remains below 500 lines and all heavy references are directly linked.
- The skill passes `quick_validate.py` and RED/GREEN behavioral regression.

## Implementation Order

1. Freeze evaluation scenarios and baseline observations.
2. Add tests for the deterministic validator and watch them fail.
3. Implement the minimal metric schema, formula catalog, and validator.
4. Write `quantitative-analysis-model.md` only for observed baseline gaps.
5. Write P0 rule cards; add P1/P2 cards only where they change a tested decision.
6. Integrate scenarios, selected cards, and metric outputs into Phase 0/1/3.
7. Update reporting standards and orchestration checks.
8. Repair both demo reports and add a validated metric example.
9. Run GREEN behavioral evaluations and validator regression.
10. Refactor wording only for observed new failures, then run full validation.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Metric schema becomes too heavy | Agents skip it or produce boilerplate | Require records only for summary-grade numeric claims |
| Rule library consumes too much context | Slower, less focused agents | Phase 0 selects and injects only relevant rule IDs |
| Static analysis appears more precise than it is | Misleading capacity decisions | Evidence kind, confidence, precision limits, calibration gate |
| Validator becomes an expression engine | Security and maintenance burden | Closed formula catalog; no arbitrary expression evaluation |
| Different agents double-count resources | Inflated totals | Stable component IDs, includes/excludes, parent/child validation |
| Scenario proliferation fragments results | Hard-to-read report | Named base scenarios and explicit derived reconciliation scenarios |
| Knowledge rules become generic C advice | Token waste | Keep only rules that alter evidence collection or cost equations |

## Open Implementation Decisions

These are implementation details, not blockers to the approved architecture:

- The initial closed formula IDs and JSON field names.
- Whether confidence labels are rendered as `C0–C3` alone or with localized text.
- Which P1 cards earn inclusion based on RED/GREEN evaluation results.
- Whether demo 2 remains a corrected positive example or also preserves a small invalid
  fixture exclusively for validator regression.
