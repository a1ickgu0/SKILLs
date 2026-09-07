# c-perf-analysis — Performance Analysis Methodology for Large C Components

A reusable SKILL for **static performance analysis** of large C/C++ codebases (≥50K lines): phased multi-agent parallel orchestration producing structured reports with file:line evidence, severity grading, quantitative reasoning, and ranked improvement priorities. Business-agnostic — applies to any multithreaded event-driven daemon-like component (protocol daemons, forwarding/session processing, media pipelines, etc.).

## Method Overview

1. **Phase 0 — Serial orientation**: a scale census sets the depth (survey/standard/deep), indexes are generated, key facts are consolidated into a checkpoint (thread model / allocator / IPC / scale constants), and the allowed read scope is determined
2. **Phase 1 — Parallel five-dimension analysis**: timers (A) / data structures & complexity (B) / memory & IPC (C) / concurrency & event loop (D) / business critical path (E); agent prompts are generated from templates
3. **Phase 2 — Appended deep-dives**: users may extend dimensions mid-run (examples: fragmentation accounting, targeted audits); letters continue sequentially and the checkpoint context is reused
4. **Phase 3 — Cross-validation & summary**: scope audit, metric reconciliation, consolidated report generation
5. **Phase 4 — Optional translation**: translation consistency contract (citation counts 1:1) + overwrite-in-place upload

## Degraded mode

When background agents are unavailable, the same methodology runs serially in a single session (see orchestration-checklist §7).

## Output Structure

```
c-perf-analysis/
├── SKILL.md                          # Skill definition (phase orchestration + failure handling + ground rules)
├── README.md                         # This file
├── references/
│   ├── orchestration-checklist.md    # Multi-agent dispatch checklist (stall recovery,
│   │                                 #   scope bounding, metric reconciliation, serial uploads, etc.)
│   ├── agent-prompt-templates.md     # Agent prompt templates for dimensions A–E
│   ├── reporting-standards.md        # Report structure, severity definitions, annotation
│   │                                 #   conventions, EN↔CN glossary, translation consistency contract
│   └── glossary.md                   # Bilingual translation-contract data
└── demo-case/                        # Open-source demonstration case (FRR bgpd, GPLv2)
    ├── README.md                     # Why FRR bgpd fits + Phase-0 style fact sheet
    └── reports/                      # Two illustrative walkthrough reports
        ├── demo_1_pipeline_teardown.md     # Dimension-E output form on real code
        └── demo_2_metric_reconciliation.md # Metric-reconciliation form (§5) in action
```

## Applicability

**Applicable**: static performance profiling of large C/C++ components, capacity/memory estimation, bottleneck localization and improvement ranking; scenarios that need multi-agent parallel coverage across multiple dimensions; daemons mixing low-level data structures + timers + IPC + concurrency.

**Not applicable**: dynamic profiling (perf/gdb runtime sampling) — this SKILL is pure static analysis; its outputs are annotated inferences backed by evidence chains. Components under 50K lines fit a single session and do not need the full orchestration.

## Field Provenance

The skill's orchestration and reporting rules were field-tested during development on real, large-scale components. This repository ships an open-source demonstration case built on FRR's bgpd — see `demo-case/` — which shows the output contracts in action against real code. The files in `references/` keep generic mentions of field experience without identifying the underlying engagement.
