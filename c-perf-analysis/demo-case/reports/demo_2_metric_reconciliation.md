# Demo Report 2 — Metric Reconciliation: Two Defensible Per-Route Memory Numbers in FRR `bgpd`

> **Purpose**: demonstrate the skill's Phase-3 **metric reconciliation** form —
> what to do when two agents report different numbers for the same quantity.
> The rule (reporting-standards §5): do NOT pick a winner; state the domain
> difference (what each count includes, which assumptions each makes), present
> a reconciled range, and cite both derivations.
>
> All evidence lines were located in the FRR snapshot (sparse checkout of
> `bgpd/` + `lib/`, 2026-08). This is an author-written illustrative walkthrough
> of the *reconciliation writing form*, using a real public case — it stands in
> for the proprietary field case that cannot be cited in the published skill.

## The conflicting claim

Two hypothetical dimension agents (B: data structures; C: memory & IPC) both
answer: **"how much memory does one BGP route cost, long-term, in `bgpd`?"**

- Agent B says: **~248 B/route**
- Agent C says: **~656 B/route**

A 2.6× gap. Neither agent is "wrong" — here is the reconciliation.

## Agent B's derivation — decision-plane, no replication, minimal extras

Per-route = one `bgp_dest` node + one selected `bgp_path_info` + attr pointer
share. `bgp_path_info` fields (`bgp_route.h:305-345`): next/prev, pi-hash
linkage, nh_thread, net back-pointer, nexthop ptr, peer ptr, from ptr, attr
ptr, extra ptr, uptime, lock, flags + more flags below line 345.

- `struct bgp_path_info` core ≈ 13 pointer/word-class fields ≈ 104 B
  [inferred: field-count × 8, assuming 64-bit; exact sizeof not compiled]
- `bgp_dest` (radix node + dest info, `bgp_table.c`) ≈ 96 B [inferred]
- allocator wrapper + bin rounding: none assumed (raw qmalloc, `memory.h:153`)
- excludes: per-subgroup adj-out, adj-in, extra struct, dampening

**Agent B total ≈ 248 B/route** (with 48 B slack for allocator rounding
[inferred]).

## Agent C's derivation — steady-state with replication, real extras

Same decision-plane core (248 B), PLUS the state a converged real deployment
actually carries:

- **adj-out per subgroup**: `struct bgp_adj_out` (`bgp_advertise.h:57-70`) —
  RB entry, subgroup ptr, TAILQ train, dest ptr, addpath_tx_id, attr ptr,
  labels ptr, adv ptr ≈ 8 words = 64 B [inferred: field-count × 8] — and the
  subgroup's adjq "represents the snapshot of every prefix advertised to the
  members" (`bgp_updgrp.h:176-179`), so it exists per (route × subgroup).
  With 2 subgroups: +128 B.
- **`bgp_path_info_extra`** (lazy, allocated when e.g. addpath/labels/MVPN
  features touch the path) ≈ 64 B [inferred; assume materialized at steady
  state].
- **attr intern share**: attributes are interned in `attrhash`
  (`bgp_attr.c:1186-1189`) so a single `struct attr` is shared — but its
  *reference* (pointer inside bgp_path_info) is already counted; C adds a
  amortized 40 B/route for distinct-attr population growth [inferred
  assumption: 1M routes, ~50K distinct attr sets × ~800 B each].
- **adj-in on receiving peers**: `struct bgp_adj_in` (`bgp_advertise.h:90+`)
  ≈ 32 B where in-bound snapshotting is active [inferred: counted at 1 subgroup
  inbound].

**Agent C total ≈ 656 B/route** (248 + 128 + 64 + 40 + 32 + slack).

## The domain difference, stated precisely

| Component of cost | Agent B | Agent C |
|---|---|---|
| bgp_dest + path_info core | ✔ | ✔ |
| alloc wrapper/bin rounding | ✘ (raw qmalloc assumed) | ✔ (48 B slack) |
| adj-out × subgroups | ✘ ("decision plane only") | ✔ (2 subgroups assumed) |
| lazy extra struct | ✘ (not materialized) | ✔ (steady-state assumption) |
| attr population amortization | ✘ (pure pointer share) | ✔ (40 B model) |
| adj-in | ✘ | ✔ (1 inbound subgroup) |

The disagreement is **scenario scope**, not arithmetic error: B answers
"minimum decision-plane footprint", C answers "converged deployment with
default replication".

## Reconciled presentation (what the summary report prints)

> **Per-route long-term memory: 250–660 B/route** depending on scenario.
> Floor (248 B): single view, no replication, lazy extras never materialized —
> derivation in agent_B report §1. Ceiling (656 B): 2 outbound subgroups +
> inbound snapshot + materialized extras + attr-population amortization —
> derivation in agent_C report §2. The largest swing factor is the
> (routes × subgroups) adj-out multiplier (`bgp_updgrp.h:176-179`,
> `bgp_advertise.h:57-70`); collapsing the two numbers requires fixing the
> subgroup count per deployment, which is configuration, not code.

## Why this form matters (the rule being demonstrated)

1. **Never silently pick one number** — a summary that prints only "656 B" or
   only "248 B" silently answers a different question than the reader asked.
2. **Decomposition table = the deliverable** — the reader picks their row.
3. **Both derivations stay cited** — the range is auditable back to each
   agent's assumptions, each marked `[inferred]` where arithmetic (not source)
   speaks.
4. **Verification hooks attach to the range, not either number** — per
   reporting-standards §5b: staged insert test (1K/10K/100K routes, measure RSS
   delta) at fixed subgroup count pins the core term; varying subgroups 1→2→4
   pins the multiplier.
