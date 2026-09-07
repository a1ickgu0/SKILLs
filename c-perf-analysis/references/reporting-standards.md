# Reporting Standards & Terminology

## 1. File naming

- Working agent outputs: `agent_<letter>_<topic>.md` (English, mechanical).
- Final consolidated report: `NN_performance_analysis.md` following the repo's docs
  numbering (check existing docs before choosing NN).
- Cloud-delivery filenames: `NN-<Chinese-name>_<English-name>.md` — meaningful, ordered,
  bilingual when the delivery audience is Chinese. Keep the local-language segment only
  when the audience requires it (e.g., `01-性能分析汇总报告_<component>-perf-summary.md`).

## 2. Evidence & annotation contract

Every claim traceable to file:line. Annotation vocabulary (translate only the tag text
when the report language changes; keep the tag present):

See [glossary.md](glossary.md) for the bilingual annotation-tag table.

## 3. Severity definitions

| severity | Criterion |
|---|---|
| High / 高 | Bounds design-point capacity (2K peers, 1M entries), O(N²)-class aggregate cost, serializes the data path, or unbounded memory risk |
| Medium / 中 | Significant in specific scenarios (churn bursts, feature enabled, multi-peer sharing); constant-factor × large-N |
| Low / 低 | Constant factors, edge cases, CLI-only paths, style/race-formality |

Every severity claim carries its evidence (file:line) in the same table row.
Severity tags are emitted in the report's language: `高/中/低` in Chinese-language
reports, `High/Medium/Low` in English-language reports.

Quantitative anchors (adjust to component scale):
- **High** — projected to consume >15% of the design-point capacity budget (CPU time
  per convergence event, memory per 1M entries), or any O(N×peers) aggregate on the
  data path.
- **Medium** — 3–15% of the budget, or large-N constant-factor costs that only
  materialize with the feature enabled.
- **Low** — <3% or non-datapath (CLI, logging, edge cases).

These anchors are calibration defaults; tighten or loosen per component in Phase 0
and record the choice in the checkpoint.

## 4. Consolidated report structure

```
# <Component> Performance Analysis Report (NN)
1. Executive summary      — architecture baseline paragraph + quantitative profile table
                            (memory/CPU at defined scenarios) + one-line top findings
2. High-severity bottlenecks    — Top-N table: # | bottleneck | dimension | evidence | impact
3. Medium-severity bottlenecks  — same format
4. Low-severity / notes         — bullet list with file:line
5. Deep-dive conclusions        — one subsection per deep-dive (F, G, …)
6. Improvement priorities       — ranked, grouped CPU / memory / IO lines; each item cites its
                            bottleneck number and expected benefit class
7. Cross-references             — agent report map (topic → file)
8. Uncertainty declaration      — assumption ledger + verification suggestions + scope statement
```

Section headings are emitted in the report's language (the layout above shows the
English form; Chinese reports use the equivalent local-language headings).

The summary cites agent reports; agent reports cite file:line. Two hops, no more.

## 5. Metric reconciliation

When agents disagree on a quantity: state the domain difference (what each count
includes, which assumptions each makes), give the reconciled range, cite both
derivations. For the worked public example, see
[demo-case/reports/demo_2_metric_reconciliation.md](../demo-case/reports/demo_2_metric_reconciliation.md).

## 5b. Verification hooks for inferred estimates

Static capacity/memory estimates (per-entry bytes, CPU per event) MUST ship with a
verification suggestion. Standard hooks:

- `malloc_usable_size()` sampling on live allocations.
- A staged allocation test: insert N=1K/10K/100K entries, measure RSS delta.
- `perf` hotspot comparison for CPU estimates.
- Cross-checking watermark arithmetic against documented comments.

The consolidated report's uncertainty declaration must list each unverified estimate
with its intended hook.

## 6. Terminology

Enforced EN↔CN table lives in [glossary.md](glossary.md) — mandatory for translation, verbatim.

## 7. Translation consistency contract (Phase 4)

> Bilingual by design: the `[inferred]` ↔ `[推断]` pairing below is contract DATA used
> to verify translated reports; it is not Chinese instruction prose.

Per translated file:
1. **Citation parity**: count `file:line`-style citations in source and output — must
   match exactly.
2. **Marker parity**: annotation-tag pairings from the bilingual table in
   [glossary.md](glossary.md) (e.g. `[inferred]` ↔ `[推断]`) must match in count;
   tags stay present.
3. **Verbatim**: identifiers, paths, numbers, units, table row counts map 1:1; no
   content added/removed/reordered.
4. **Terminology**: §6 table mandatory; translator reports uncertain terms in its
   summary reply.
5. **Independent re-verification**: the orchestrator (not the translator) re-checks
   parity counts before delivery.

## 8. Cloud delivery notes

- Serial uploads to one target folder (server-side concurrent-conflict risk).
- Overwrite existing remote files via file token, not new uploads.
- Rename remote files to §1's `NN-<Chinese-name>_<English-name>` scheme at or after upload.
- Offer import-as-online-doc conversion when collaboration/comments are expected;
  raw `.md` upload suffices for archival.
