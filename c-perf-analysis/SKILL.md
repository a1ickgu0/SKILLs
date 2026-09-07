---
name: c-perf-analysis
description: Static performance analysis of large C/C++ components (50K+ lines) using phased multi-agent orchestration. Covers timers/periodic tasks, data structures & algorithm complexity, memory & IPC overhead, concurrency/locks/event loops, and the business critical path. Use when the user asks for performance profiling, bottleneck hunting, or capacity/memory analysis of a large C codebase such as a multithreaded event-driven daemon, and expects a structured report with file:line evidence. Use this skill for static performance analysis and bottleneck localization of large C components.
---

# C Performance Analysis — SKILL

## Purpose

Operationalize a battle-tested methodology for static performance analysis of large C/C++
components. The workflow: serial orientation → parallel multi-dimension agent analysis →
cross-validation → consolidated report. It is business-agnostic: dimension E adapts to
whatever the component's critical path is (route convergence, packet forwarding, request
handling, media processing, etc.).

## Language Policy

SKILL instructions are in English. Report and summary output language follows the user's
input language (Chinese input → Chinese report; English → English). Never translate file
paths, identifiers, or numeric evidence. For translation consistency rules see
[references/reporting-standards.md](references/reporting-standards.md).

## Scope & Prerequisites

Target: C/C++ components ≥ 50K lines (below that, a single-session analysis suffices —
borrow Phases 0/3 only).

Before dispatching anything, confirm with the user or discover:
1. **Source directory** of the component.
2. **Allowed read scope**: component source + any SDK/dependency header directories the
   user authorizes. Everything else is out of bounds. Record this list — every agent
   prompt will embed it verbatim.
3. **Build config** to locate include paths and feature flags.
4. **Encoding**: run `file <representative-file>` on a few sources. If GBK/ISO-8859 is
   detected, EVERY grep in every agent prompt must carry `-a` (text-mode), or grep
   silently returns nothing. This warning is mandatory in all templates.
5. **User focus** (optional): named subsystems or concerns become mandatory Phase 2 targets.

## Phase 0 — Index & Orientation (serial, mandatory)

All later phases depend on this. Do not parallelize.

1. **Scale & depth**: count `.c`/`.h` files and total lines.
   | Total lines | Depth | Coverage |
   |---|---|---|
   | < 50K | deep | all phases, full detail |
   | 50K–150K | standard | phases 0–3, abbreviated deep-dives |
   | > 150K | survey | phases 0–3 + hotspot-style deep-dives only |

   Treat the line-count depth as a floor, not a ceiling — if Phase 0 probing finds high
   structural concurrency (e.g. >8 real threads, or >4 IPC channels, or >20 distinct
   periodic timers), promote the depth one level (standard→deep, deep stays deep) and
   record the promotion reason in the checkpoint.
2. **Index files** into an output dir (default `docs/analysis/`):
   - `index_files.csv` — file inventory sorted by size (`find … | xargs wc -l`)
   - `index_includes.csv` — include dependency map (grep `#include` with edge format `file → header`)
3. **Probe key facts** for the checkpoint: process/thread model (grep `pthread_create`,
   `thread_add_*`, `epoll`/`poll`/`select`), timer infrastructure, allocator entry points
   (`XMALLOC`/`malloc` wrappers, memory pools), IPC framework, largest structs, scale
   constants (`*_MAX` defines).
4. **Read the top 3–5 largest headers** for semantic enrichment of core data structures.
5. **Write `analysis_checkpoint.md`** (English, machine-readable): scale numbers, depth,
   key facts with file:line, allowed directories, per-phase completion status. Every
   parallel agent receives the relevant subset as `{CHECKPOINT_FACTS}` — this prevents
   duplicate exploration.
6. **Task board**: create tasks (TaskCreate) for dimensions A–E plus summary. Mark in
   progress/completed as agents launch/finish.

### Dimension fact-flow map

Known fact dependencies between dimensions: C (memory/IPC) feeds ad-hoc allocator
deep-dives (F/G); D (concurrency) findings may change aggregate-cost conclusions in A
and E; B's per-entry sizing feeds C and E capacity estimates. Therefore: per-agent
completion status goes into `analysis_checkpoint.md` incrementally as each agent
finishes (crash safety); the key quantitative facts from all agent reports get one
batched refresh into the checkpoint after ALL of Phase 1 finishes, before Phase 2
concern additions and Phase 3 reconciliation consume them.

## Phase 1 — Parallel Dimension Analysis

Dispatch dimensions A–E **in parallel** (single message, multiple background Agent calls).
Prompts are generated from [references/agent-prompt-templates.md](references/agent-prompt-templates.md).
Each prompt MUST embed, in this order:

1. Role + output file path (`agent_<X>_<topic>.md`, English, tables, ≤400 lines)
2. **Scope block**: allowed directories verbatim + "do NOT read anything else; if a
   definition lives outside, note `[external, not read]`"
3. **Encoding warning**: emitted via the templates' conditional `{ENCODING_WARNING}`
   placeholder — only when the Phase 0 probe (`file -bi`) finds a non-UTF-8 charset
   (e.g., GBK/ISO-8859 → "ALWAYS use `grep -a`"). Otherwise omit.
4. `{CHECKPOINT_FACTS}`: the Phase 0 facts relevant to that dimension (with file:line)
5. Dimension-specific tasks (from template), each claim requiring file:line citation,
   inferences marked `[inferred]`
6. Output format contract (sections + tables + severity-with-evidence bottleneck list)
7. Reporting constraint: reply in under 150 words

Output convention: each agent writes `agent_<letter>_<topic>.md` in the output dir.

### Dimension board

| Dim | Focus | Template |
|---|---|---|
| A | Timers & periodic tasks: inventory, per-expiry cost, scan-type full-table walks, timer storms | T-A |
| B | Data structures & complexity: struct inventory, operation complexity, O(N²)/O(N×M) risks, per-entry memory | T-B |
| C | Memory & IPC: allocator/pool inventory, per-entry allocation count, IPC copy chain, HA/backup multipliers, watermarks | T-C |
| D | Concurrency, locks & event loop: thread/channel inventory, loop walkthrough, critical-section audit, cross-thread signals, backpressure | T-D |
| E | Business critical path: end-to-end pipeline, per-stage cost, batching/quota, per-peer replication, capacity estimate | T-E |

Operational rule for dimension E: confirm the critical path with the user when (a) the
entry-point fan-out is ambiguous (two or more plausible ingress paths of similar
weight), or (b) source comments/README contradict each other about the main processing
path. Otherwise proceed with the best-evidence path and note the assumption.

## Phase 2 — Optional Deep-Dive (user-triggered or hotspot-driven)

Users may append concerns mid-analysis (e.g., "analyze fragmentation under our jemalloc",
"audit the checksum path"). Each new concern becomes a new agent with a continued letter
(F, G, …), a TaskCreate entry, and a prompt that includes: the original scope block +
encoding warning + the now-extended `{CHECKPOINT_FACTS}` (including Phase 1 findings
relevant to the new concern — do not make the agent re-derive them). No canned template
exists for ad-hoc concerns: decompose the request into a named task block following the
T-A…T-E output conventions (sections, tables, severity+evidence), and confirm the
decomposition with the user when the ask is novel.

Timing vs. Phase 3: if a deep-dive is still running when Phase 3 has already started,
the orchestrator has two legitimate options: (1) hold the summary until the deep-dive
finishes, informing the user of the expected delay; or (2) deliver an interim summary
marked as such, then issue a numbered addendum when the deep-dive completes. Either way
the summary's `[not analyzed]` list must reference the pending deep-dive explicitly.

## Phase 3 — Cross-Validation & Summary

1. **Scope check**: for each `agent_*.md`, grep for absolute paths or directory names
   outside the allowed set. Fix or annotate violations.
2. **Metric reconciliation**: when two agents report different numbers for the same
   quantity (e.g., bytes per entry), do NOT pick a winner — clarify the domain difference
   (what each count includes, which assumptions each makes) and present a reconciled
   range in the summary, citing both derivations.
3. **Consolidated report** `NN_performance_analysis.md` (numbered to follow repo docs
   convention; user language) following the structure in
   [references/reporting-standards.md](references/reporting-standards.md):
   executive summary → top high-severity bottlenecks → medium → low → deep-dive
   conclusions → ranked improvements (CPU / memory / IO lines) → cross-references →
   uncertainty declaration.
4. **Update the checkpoint** with final document list, reconciliations performed, and
   unresolved gaps.

## Phase 4 — (Optional) Translation & Delivery

If the user requests translated reports (e.g., Chinese versions for upload to a doc
platform): dispatch one translation agent per report, in parallel, with the consistency
contract from [references/reporting-standards.md](references/reporting-standards.md)
(citation count 1:1, terminology table from [references/glossary.md](references/glossary.md)
mandatory, identifiers/numbers verbatim, marker parity per the bilingual tag table).
Then independently re-verify citation counts and spot-check
key numbers before delivery. When uploading to cloud drives: overwrite-in-place
(file token) rather than new uploads; serial uploads to the same target folder (parallel
writes to one target can hit server-side conflicts).

## Failure Handling

See [references/orchestration-checklist.md](references/orchestration-checklist.md) for
the full protocol. Quick reference:

- **Proactive stall check**: if the harness has no stall watchdog, poll each agent's
  output file mtime every ~10 minutes; a file with no mtime change across two
  consecutive polls is suspected stalled and enters the resume protocol.
- **Agent stall** (no progress watchdog fires): check whether partial output was written →
  resume with "write what you have first, then refine" → if the same agent stalls
  again, take over the dimension in the main session using the checkpoint facts
  (remaining work is usually 1–2 greps plus writing).
- **Stream errors** (`API Error: Stream error`): check if the output file was written;
  resume only if not.
- **Report the scope**: any analysis not completed (agent died, dimension skipped) is
  listed under `[not analyzed]` in the summary — never silently dropped.

## Ground Rules

- **Analysis only — never modify source code.** Improvement proposals go in the report,
  ranked by expected benefit, with evidence.
- Every claim must be traceable to file:line. Mark inferences `[inferred]`; never
  fabricate semantics from naming alone.
- Respect checkpoint continuity: on session restart, read `analysis_checkpoint.md` first.
- Scale determines pace: a 300K-line component needs the survey posture — index-level
  descriptions first, expand to code excerpts only where a bottleneck demands it.
