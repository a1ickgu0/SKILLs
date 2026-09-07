# Demo Case — Static Performance Analysis of FRR `bgpd`

This directory is the **public, open-source demonstration case** for the
c-perf-analysis skill. It replaces the original field case (a proprietary
networking component) that cannot be referenced in the published skill.

## Why FRR `bgpd`

| Criterion | Fit |
|---|---|
| License | GPLv2 — free to cite, excerpt, and redistribute references to |
| Domain | A real, production-grade BGP daemon — matches the skill's origin domain (route convergence, T-E template: "BGP → route convergence") |
| Architecture | Event-driven daemon over a cooperative scheduler (`lib/` event/thread framework), no pthread-per-peer — exactly the "pseudo-thread" pattern the skill's dimension D targets |
| Structures | Per-destination route tables over radix trees (`bgpd/bgp_table.c`), per-path info structs — matches dimension B |
| IPC | ZAPI stream-based IPC to the Zebra RIB (`bgpd/bgp_zebra.c`) — matches dimension C's IPC copy-chain audit |
| Timers | Keepalive/hold/GR-stale/advertisement-delay timers over `event_add_timer*` (`bgpd/bgp_fsm.c`) — matches dimension A |
| Scale | ~184K lines in `bgpd/` alone (≥50K threshold met; falls in the skill's "standard/survey" band) |

Source snapshot used: FRRouting/frr, sparse checkout of `bgpd/` + `lib/`
(shallow clone, August 2026). Line counts below refer to that snapshot.

## Phase-0 style facts (demonstration of a checkpoint's key-facts slice)

- **Scale**: 179 C/H files under `bgpd/` (~184K lines), plus `lib/` (~188K lines).
  Largest: `bgpd/bgp_vty.c` (26,357), `bgpd/bgp_route.c` (20,624),
  `bgpd/bgpd.c` (10,365), `bgpd/bgp_evpn.c` (9,213), `bgpd/bgp_routemap.c` (8,813).
- **Thread model**: cooperative event scheduler; sessions ride `struct event`
  registrations (`event_add_read/write/timer*`), no thread-per-peer.
- **Timers**: `bgpd/bgp_fsm.c` arms per-connection FSM timers
  (`event_add_timer`), e.g. GR stale `bgp_fsm.c:232`, update-generation
  `bgp_fsm.c:605`, max-med on-startup `bgp_fsm.c:1312`.
- **Allocator**: `lib/memory.h:153-154` — `XMALLOC/XCALLOC(mtype, size)` wrap
  `qmalloc/qcalloc` (mtype-tagged malloc); pools/allocators are mtype-registered.
- **IPC**: ZAPI client (`bgpd/bgp_zebra.c:63-64`), stream-based message layer
  (100 `stream_*` call sites in `bgp_zebra.c`).
- **Encoding**: UTF-8 clean (`file` probe on largest sources) — the skill's
  `grep -a` warning is NOT emitted for this case (demonstrates the conditional
  `{ENCODING_WARNING}` placeholder).

## Contents

```
demo-case/
├── README.md                              # this file
└── reports/
    ├── demo_1_pipeline_teardown.md        # dimension-E style walkthrough (T-E output form)
    └── demo_2_metric_reconciliation.md    # how to reconcile two agents' numbers (§5 form)
```

## What these demo reports show

1. **demo_1** — the shape of a single dimension agent's output: sections, a
   per-stage table, file:line citations, an `[inferred]` estimate with visible
   arithmetic, and a severity+evidence bottleneck list.
2. **demo_2** — the metric-reconciliation form: two defensible derivations of
   the same quantity, the domain difference spelled out, and a reconciled range
   citing both — the skill's "never silently pick a winner" rule in action.

Both are *illustrative walkthroughs by the skill author* (evidence lines were
actually located in the FRR snapshot), not full parallel-agent runs; they are
scoped excerpts meant to teach the output contracts, not exhaustive analyses.

## Regenerating the snapshot

```sh
git clone --depth 1 --filter=blob:none --sparse https://github.com/FRRouting/frr.git
cd frr && git sparse-checkout set bgpd lib
```
