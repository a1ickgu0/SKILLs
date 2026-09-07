# Demo Report 1 — Dimension-E Walkthrough: BGP Route Convergence Pipeline in FRR `bgpd`

> **Purpose**: an illustrative, author-written walkthrough of what ONE dimension
> agent's output looks like under this skill (T-E template: business critical
> path → route convergence). It demonstrates the output contract — sections,
> tables, file:line citations, `[inferred]` marking with visible arithmetic,
> severity+evidence bottleneck list — on a real open-source codebase (FRR,
> GPLv2). It is NOT a full parallel-agent analysis; scope is deliberately a
> selected excerpt.
>
> Snapshot: FRRouting/frr sparse checkout (`bgpd/` + `lib/`), 2026-08.
> All line numbers refer to that snapshot and MUST be re-verified before reuse
> (the repo moves).

## 1. End-to-end pipeline (text diagram, stage → file:line)

```
[ingress]  socket read events (event_add_read) ................ bgpd/bgp_io.c
             └─ per-connection FIFO of pending peer connections .. bm->connection_fifo,
                                                                   referenced bgp_route.c:5174
    ↓
[decode/FSM] message decode + session state machine ........... bgpd/bgp_packet.c, bgpd/bgp_fsm.c
    ↓
[enqueue]   destinations marked dirty into per-BGP MetaQ
             (process_queue = work_queue_new) ................. bgp_route.c:5387
             meta_queue_add / early_route_meta_queue_add ...... bgp_route.c:5192
    ↓
[decision]  bestpath per destination, one item per run:
             bgp_process_main_one() ........................... bgp_route.c:4333
             scheduled via meta_queue_process() ............... bgp_route.c:5161
    ↓
[replicate] update groups → subgroups coalesce peers with the
             same outbound policy; per-subgroup adj-out RB tree
             (snapshot of every advertised prefix) ............ bgp_updgrp.h:164,177-179;
                                                                bgp_advertise.h:57
    ↓
[build]     shared packet encode into bpacket queue, then
             per-peer reformat: bpacket_reformat_for_peer() ... bgp_packet.c:616-619
    ↓
[egress]    per-connection write events; MRAI/routeadv timer
             re-arms generation ............................... bgp_fsm.c:593-608, bgp_io.c
```

## 2. Per-stage cost — top candidates

| # | Stage | Cost driver | Evidence | Class |
|---|---|---|---|---|
| 1 | bestpath selection | per-dest comparison set (attr cmp, IGP metric) | `bgp_process_main_one` bgp_route.c:4333; `attrhash_cmp` bgp_attr.c:1130 | O(N) aggregate |
| 2 | route-map/policy evaluation at update build | full policy engine (`bgp_routemap.c` ≈ 8.8K lines) | bgp_routemap.c (file scale) | per (dest × subgroup) |
| 3 | attr interning | hash key + cmp per attribute set | `attrhash_key_make` bgp_attr.c:1069; `bgp_attr_intern` bgp_attr.c:1320 | per path learn |
| 4 | radix table ops | insert/lookup/delete on `bgp_table.c` route nodes | bgp_table.c (whole file) | O(len(prefix)) |
| 5 | adj-out maintenance | RB insert/delete per (dest × subgroup) | `struct bgp_adj_out` bgp_advertise.h:57 (RB_ENTRY) | O(log N) each |

## 3. Deferral & batching (the interesting part)

- **Decision is deferred, not inline**: ingestion marks destinations dirty and a
  `work_queue` drains them (`meta_queue_process`, bgp_route.c:5161) — burst
  absorption at the cost of convergence latency.
- **Explicit yield/backpressure heuristic**: if >10 peers sit on the connection
  FIFO, MetaQ yields to packet processing, but runs anyway every 10th run to
  avoid starvation — `bgp_route.c:5174-5181`, returning `WQ_QUEUE_BLOCKED` /
  `WQ_REQUEUE` (bgp_route.c:5189). Constants (10, 10) are static — a fairness-
  vs-throughput tuning knob [inferred: no config surface found in scope].
- **Update generation is timer-deferred**: MRAI/routeadv expiry schedules
  `bgp_generate_updgrp_packets` with 0 ms delay — `bgp_fsm.c:604` — i.e., the
  timer acts as a batching barrier; comment notes MRAI restarts when the FIFO
  is built (bgp_fsm.c:607-608).

## 4. Replication economics

- **Grouping**: peers sharing outbound policy join an `update_subgroup`
  (`bgp_updgrp.h:164`); the subgroup owns one `bpacket_queue` and one adjq —
  the comment at `bgp_updgrp.h:177-180` states the adj queue "represents the
  snapshot of every prefix that has been advertised to the members".
- **Shared encode, per-peer reformat**: packets are encoded once per subgroup,
  then `bpacket_reformat_for_peer` adapts per peer (`bgp_packet.c:616-619`).
  What breaks grouping: per-peer outbound policy differences split subgroups,
  multiplying adj-out storage [inferred from structure].
- **Per-subgroup state per prefix**: `struct bgp_adj_out` (bgp_advertise.h:57-70)
  — RB linkage, subgroup pointer, TAILQ train, dest pointer, addpath_tx_id,
  attr pointer, labels, adv pointer. See Demo Report 2 for its byte math.

## 5. Capacity estimate `[inferred]` — arithmetic shown

Scenario: 1M IPv4 unicast routes, 1 view, 2 subgroups, bestpath recompute of a
full table (worst-case churn burst).

- Decision: assume 5–15 µs per destination for `bgp_process_main_one` work
  (comparisons + radix ops + attr cmp) → 1M × (5–15 µs) ≈ **5–15 s CPU**,
  wall-clock longer with the >10-peer yield rule interleaving packet work
  (bgp_route.c:5178-5181) [inferred; constant is an assumption, not measured].
- Replication memory multiplier: 1M × 2 subgroups × ~80 B adj-out ≈ **160 MB**
  on top of decision-plane storage (derivation in Demo Report 2).
- Dominant term at scale: per-(dest × subgroup) policy evaluation + RB
  maintenance, not decode.

## 6. Bottleneck list (severity + evidence)

| Severity | Bottleneck | Evidence |
|---|---|---|
| High | Adj-out memory scales as O(prefixes × subgroups); with many outbound policy splits the multiplier approaches per-peer | bgp_updgrp.h:177-180; bgp_advertise.h:57-70; §5 arithmetic [inferred] |
| Medium | Static MetaQ yield constants couple convergence latency to peer FIFO depth with no adaptation | bgp_route.c:5174-5181 |
| Medium | attr intern hash created without explicit initial size → rehash growth at high distinct-attr counts | bgp_attr.c:1186-1189 (`hash_create` default) [inferred] |
| Low | Every-10th-run starvation escape is a magic constant that silently shapes tail latency | bgp_route.c:5178-5181 |

## Scope & uncertainty declaration

- Excerpt scope: `bgpd/` only; `lib/` event framework and Zebra/ZAPI internals
  were not walked end-to-end here (`[external, not read]` for this demo).
- The 5–15 µs per-dest constant is an assumption — verification hook per
  reporting-standards §5b: perf/trace sampling of `bgp_process_main_one` under
  a 100K-route replay, plus a staged insert test for memory terms.
