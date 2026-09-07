# Orchestration Checklist — Multi-Agent Performance Analysis

Field-tested rules from a large-scale component analysis (5 parallel dimension agents +
1 user-appended deep-dive + translation pipeline). Order follows a real run's timeline.
The public demonstration case in `demo-case/` (FRR bgpd) shows the output contracts
these rules produce.

## 1. Before dispatch (Phase 0)

- [ ] **Serial-first**: Phase 0 index MUST complete before any agent launches. All
      parallel agents depend on the checkpoint facts.
- [ ] **Index only source files**: `find <src> \( -name '*.c' -o -name '*.h' \) -type f
      | xargs wc -l | sort -n`. Write the head of the result (Top 10 largest files)
      into the checkpoint for all agents to reuse; ignore non-source files (build
      dirs, vendored code, generated files) to keep the line counts honest.
- [ ] **Freeze the checkpoint**: write verified facts (thread model, allocator entry,
      IPC framework, scale constants, key structs) with file:line into
      `analysis_checkpoint.md`. This is the anti-duplication asset — each agent gets
      only the slice it needs.
- [ ] **Explicit scope block**: list allowed directories verbatim in every prompt +
      the sentence "do NOT read anything else". Vague scope ("explore the codebase")
      invites agents to wander into sibling repos.
- [ ] **Encoding probe**: `file` a few sources. GBK/ISO-8859 → inject "ALWAYS use
      `grep -a`" into every prompt. Silent-empty greps have burned real runs: the
      agent concludes "pattern not found" and misreports.
- [ ] **Task board**: TaskCreate one entry per dimension + summary; mark in_progress
      at dispatch, completed only when the report file exists AND is scope-checked.
- [ ] **Include-map noise caveat**: grep-based include maps are text-level
      approximations — they include conditional-compilation noise (`#if 0` blocks,
      platform branches). Annotate the map as approximate in the checkpoint, or strip
      obvious dead branches before sharing.

## 2. During dispatch

- [ ] **Parallel launch**: send all dimension agents in a single message (background).
      Templates guarantee uniform output contracts; don't hand-roll per-dimension
      prompts from scratch.
- [ ] **150-word report constraint**: forces agents to return conclusions, not
      transcripts. The full evidence lives in their output file.
- [ ] **Do not duplicate agent work**: once a dimension is dispatched, the main
      session must not explore the same files for it. Work only on orchestration.
- [ ] **Incremental checkpoint updates (status) + batch fact refresh (Phase 1 end)**:
      after each dimension agent finishes (report file on disk + scope-checked),
      immediately update its completion status in analysis_checkpoint.md — do not
      batch status updates to Phase 3, or a mid-run crash loses the trail. Separately,
      once ALL of Phase 1 is done, do one batched write refreshing the key
      quantitative facts from every agent report into the checkpoint (SKILL.md
      Phase 0 "Dimension fact-flow map") before Phase 2/3 consume them.

## 3. Failure handling protocol

- [ ] **Stall (watchdog)**: the harness kills agents at ~600s of no progress.
      1. Check if the output file exists / has partial content.
      2. Resume via SendMessage with: "continue; write what you have to the file
         FIRST, then refine" — forces disk persistence before further exploration.
      3. Second stall on the same agent → take over in the main session. You have
         the checkpoint facts; the remaining work is usually 1–2 greps plus writing.
      4. If the harness lacks a stall watchdog, poll each agent's output-file mtime
         every ~10 minutes; no mtime change across two consecutive polls = suspected
         stall, enter the resume protocol above.
- [ ] **Stream errors** (`API Error: Stream error: error decoding response body`):
      check the output file; if absent, resume the agent. If present, treat as done
      and verify.
- [ ] **Never mark a dimension complete without its report file on disk.** An agent
      "finished" notification with no file = failure.

## 4. Cross-validation (Phase 3)

- [ ] **Scope audit**: grep each `agent_*.md` for absolute paths / out-of-scope
      directory names. Agents occasionally cite what they read despite instructions.
      Fix (re-run) or annotate before the summary.
- [ ] **Metric reconciliation**: conflicting numbers between agents (e.g., two
      per-entry byte totals 2× apart) are usually domain differences, not errors:
      one includes replication/lazy state, one assumes a minimal decision-plane
      footprint. Present a reconciled range with both derivations cited. Never
      silently pick one. See `demo-case/reports/demo_2_metric_reconciliation.md`
      for the worked form.
- [ ] **Assumption ledger**: collect every `[inferred]` and every external-assumption
      (e.g., "jemalloc size-class behavior", "no wrapper header") into the summary's
      uncertainty declaration, each with a concrete verification suggestion
      (malloc_usable_size, staged test, etc.).

## 5. User-appended concerns mid-run

- [ ] **New dimension = new agent + continued letter** (F, G, …) + new task entry.
- [ ] **Extend `{CHECKPOINT_FACTS}`**: feed the new agent the Phase 1 findings
      relevant to its concern (e.g., for a fragmentation deep-dive: allocator entry,
      pool config, per-entry allocation counts from dimension C). Do not let it
      re-derive known facts.
- [ ] **Re-plan before re-dispatch**: for a vague user request ("analyze memory from
      fragmentation angle"), decompose into a named framework (internal frag → wrapper
      overhead → external frag → double-cache → aggregate scenarios) BEFORE launching;
      confirm the framework with the user when the ask is novel.

## 6. Delivery

- [ ] **Serial uploads to one target**: batch import/upload to the same cloud-drive
      folder MUST be serial — parallel writes to one target can hit server-side
      concurrent-conflict errors.
- [ ] **Overwrite, don't duplicate**: when replacing an existing remote file, upload
      with the existing file token (overwrite semantics), not as a new file.
- [ ] **Meaningful remote filenames**: rename on upload/after upload to
      `NN-<Chinese-name>_<English-name>.ext` (or repo convention) — keep the local
      language part only when the delivery audience requires it. Raw agent filenames
      (`agent_A_timers.md`) are meaningless to stakeholders.
- [ ] **Translation consistency**: per-file citation-count parity (source vs
      translated) is the cheap, objective check — verify 1:1 on every file. Plus
      terminology-table enforcement (see glossary.md).

## 7. Degraded single-session mode

- [ ] **When to use**: background agents are unavailable (no Agent tool, or a
      restricted environment). Run the same methodology serially in one session.
- [ ] **Phase 0 unchanged**: the checkpoint is even more valuable here — it serves as
      your own working memory across a long serial run.
- [ ] **Execute dimensions A→E one at a time**: reuse the same templates and output
      contracts; the `agent_<X>.md` files are still written to the output dir.
- [ ] **Phase 3 unchanged**: cross-validation and the summary flow are identical.
- [ ] **Expect ~5× wall-clock**: scope discipline and the evidence contract matter
      MORE, not less — there are no parallel agents to cross-check each other.
