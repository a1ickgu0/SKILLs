# c-perf-analysis Quantitative Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a scenario-bound, evidence-graded, mechanically validated quantitative-analysis layer to `c-perf-analysis` without weakening its existing static-analysis and multi-agent workflow.

**Architecture:** Keep `SKILL.md` as the orchestration entry point. Put formulas/metric records and software-performance rule cards in two directly linked references, emit one JSON metrics file per dimension agent, and validate those records with a standard-library Python script before Phase 3 reconciliation. Existing Markdown reports remain the human-readable deliverable.

**Tech Stack:** Markdown skill instructions, JSON metric records, Python 3 standard library (`json`, `decimal`, `pathlib`, `unittest`), Git.

**Spec:** `docs/superpowers/specs/2026-09-01-c-perf-analysis-quantitative-model-design.md`

## Global Constraints

- The analyzed C/C++ component remains read-only; this skill never modifies target source.
- Static-only analysis must not invent CPU time, utilization, throughput, exact RSS, stable queue wait, lock wait, or latency percentiles.
- Every summary-grade numeric claim requires a scenario-bound metric record with typed units, provenance, evidence kind, confidence, includes/excludes, and a verification hook.
- Evidence kinds are `SF`, `SB`, `EA`, and `DC`; confidence values are `C0` through `C3`.
- Metric files are independent `agent_<letter>_metrics.json` files; parallel agents never write one shared metrics file.
- The validator uses only the Python standard library and never evaluates arbitrary expressions.
- Heavy references are linked directly from `SKILL.md`; do not create a nested runtime-reference hierarchy.
- `SKILL.md` stays below 500 lines.
- Preserve the existing language policy, scope restriction, encoding warning, file:line evidence contract, and `[not analyzed]` disclosure.
- Existing unrelated worktree changes must not be staged or modified.

## File Map

| File | Responsibility |
|---|---|
| `c-perf-analysis/scripts/validate_metrics.py` | Closed-catalog validation for metric JSON files |
| `c-perf-analysis/scripts/tests/test_validate_metrics.py` | Standard-library unit tests for validator behavior |
| `c-perf-analysis/scripts/tests/fixtures/*.json` | Valid and intentionally invalid metric documents |
| `c-perf-analysis/references/quantitative-analysis-model.md` | Scenario, units, formulas, evidence, metric schema, validation contract |
| `c-perf-analysis/references/software-performance-rules.md` | C, data-structure, design-pattern, and coding-experience rule cards |
| `c-perf-analysis/SKILL.md` | Phase routing, scenario/rule-card selection, metric validation gate |
| `c-perf-analysis/references/agent-prompt-templates.md` | Per-agent quantitative output contract and A–E allowed outputs |
| `c-perf-analysis/references/reporting-standards.md` | Rendering only validated metrics and separating metric semantics |
| `c-perf-analysis/references/orchestration-checklist.md` | Phase 0/1/3 mechanical gates |
| `c-perf-analysis/references/glossary.md` | Stable bilingual terms for new evidence/metric vocabulary |
| `c-perf-analysis/demo-case/evaluations/quantitative-scenarios.md` | RED/GREEN skill behavior cases and scoring rubric |
| `c-perf-analysis/demo-case/reports/demo_1_pipeline_teardown.md` | Corrected CPU/work-unit and replication-range demonstration |
| `c-perf-analysis/demo-case/reports/demo_2_metric_reconciliation.md` | Arithmetically closed reconciliation demonstration |
| `c-perf-analysis/demo-case/reports/demo_metrics.json` | Machine-valid demo metric records |
| `c-perf-analysis/README.md` | Updated method and output-tree overview |

---

### Task 1: Closed-Catalog Metric Validator

**Files:**
- Create: `c-perf-analysis/scripts/validate_metrics.py`
- Create: `c-perf-analysis/scripts/tests/test_validate_metrics.py`
- Create: `c-perf-analysis/scripts/tests/fixtures/valid-memory.json`
- Create: `c-perf-analysis/scripts/tests/fixtures/invalid-arithmetic.json`
- Create: `c-perf-analysis/scripts/tests/fixtures/invalid-ea-hook.json`

**Interfaces:**
- Consumes: JSON object with top-level `schema_version`, `scenario`, and `metrics`.
- Produces: `validate_document(document: dict) -> list[str]` and `validate_file(path: pathlib.Path) -> list[str]`.
- CLI accepts one or more file arguments: `python c-perf-analysis/scripts/validate_metrics.py FILE [FILE2]`; exit `0` when all pass, exit `1` and print `<path>: <metric_id>: <error>` for failures.
- Initial formula IDs: `MEM.INSTANCE_PRODUCT` and `MEM.COMPONENT_SUM`.

- [ ] **Step 1: Write fixture-driven failing unit tests**

```python
class ValidateMetricsTests(unittest.TestCase):
    def test_valid_memory_range_passes(self):
        self.assertEqual(validate_file(FIXTURES / "valid-memory.json"), [])

    def test_arithmetic_mismatch_names_metric(self):
        errors = validate_file(FIXTURES / "invalid-arithmetic.json")
        self.assertTrue(any("MEM.ROUTE.TOTAL" in error and "512" in error for error in errors))

    def test_experience_assumption_requires_verification_hook(self):
        errors = validate_file(FIXTURES / "invalid-ea-hook.json")
        self.assertTrue(any("verification_hook" in error for error in errors))
```

- [ ] **Step 2: Run tests and verify RED**

Run:

```bash
python -m unittest discover -s c-perf-analysis/scripts/tests -v
```

Expected: import or missing-file failure because `validate_metrics.py` does not exist.

- [ ] **Step 3: Implement interval arithmetic and document validation**

Implement the exact public signatures `validate_document(document: dict) -> list[str]`,
`validate_file(path: Path) -> list[str]`, and
`main(argv: list[str] | None = None) -> int`.

Use `Decimal(str(value))` for all arithmetic. `MEM.INSTANCE_PRODUCT` multiplies the intervals of inputs with roles `base_count`, `fanout`, `occupancy`, and `size`; omitted occupancy defaults to the exact literal `1`. `MEM.COMPONENT_SUM` adds referenced component intervals. Reject unknown formula IDs, missing required fields, mismatched result units, arithmetic mismatch, invalid evidence/confidence values, and missing verification hooks for `EA` or `DC`-dependent predictions.

- [ ] **Step 4: Populate exact fixtures**

`valid-memory.json` models:

```text
1,000,000 routes × 2 subgroups/route × 64–80 B/adj_out
= 128,000,000–160,000,000 B
```

`invalid-arithmetic.json` declares components `248 + 128 + 64 + 40 + 32 B` but result `656 B`; expected recomputed value is `512 B`.

`invalid-ea-hook.json` is otherwise valid but has `evidence.kind = "EA"` and an empty `verification_hook`.

- [ ] **Step 5: Run focused tests and CLI smoke tests**

```bash
python -m unittest discover -s c-perf-analysis/scripts/tests -v
python c-perf-analysis/scripts/validate_metrics.py c-perf-analysis/scripts/tests/fixtures/valid-memory.json
python c-perf-analysis/scripts/validate_metrics.py c-perf-analysis/scripts/tests/fixtures/invalid-arithmetic.json
```

Expected: all unit tests PASS; valid fixture exits `0`; invalid fixture exits `1` and reports `MEM.ROUTE.TOTAL` with recomputed `512 B`.

- [ ] **Step 6: Commit the validator slice**

```bash
git add c-perf-analysis/scripts
git commit -m "feat: validate performance metric records"
```

---

### Task 2: Shared Quantitative Analysis Reference

**Files:**
- Create: `c-perf-analysis/references/quantitative-analysis-model.md`
- Modify: `c-perf-analysis/scripts/tests/test_validate_metrics.py`
- Modify: `c-perf-analysis/scripts/validate_metrics.py`

**Interfaces:**
- Consumes: Phase 0 scenario facts and dimension-agent observations.
- Produces: scenario schema, semantic-unit rules, formula catalog, evidence/confidence rules, metric-record schema, and validation checklist used by Tasks 4–7.
- Extends validator formula IDs with `RATE.EVENT_WORK` and `QUEUE.STABILITY` only after failing tests exist.

- [ ] **Step 1: Add failing tests for rate and queue formulas**

```python
def complete_metric(overrides):
    metric = {
        "metric_id": "TEST.METRIC",
        "name": "test metric",
        "dimension": "cpu",
        "scenario_ref": "test.scenario.v1",
        "evidence": {"kind": "SB", "confidence": "C2"},
        "scope": {"includes": ["test component"], "excludes": ["other work"]},
        "assumptions": [],
        "verification_hook": {"method": "compare with a focused runtime counter"},
    }
    metric.update(overrides)
    return metric

def make_document(metric):
    return {
        "schema_version": "1.0",
        "scenario": {"scenario_id": "test.scenario.v1", "workload_mode": "steady"},
        "metrics": [metric],
    }

def test_event_work_rate_multiplies_rate_by_work_per_event(self):
    document = make_document(complete_metric({
        "metric_id": "CPU.HASH.PROBES",
        "formula": {
            "id": "RATE.EVENT_WORK",
            "inputs": [
                {"role": "event_rate", "value": 200000, "unit": "count(event)/s"},
                {"role": "work_per_event", "value": 978.5625,
                 "unit": "count(hash_probe)/count(event)"},
            ],
            "output_unit": "count(hash_probe)/s",
        },
        "result": {"value": 195712500, "unit": "count(hash_probe)/s"},
    }))
    self.assertEqual(validate_document(document), [])

def test_queue_wait_is_rejected_when_rho_is_not_below_one(self):
    metric = complete_metric({
        "metric_id": "QUEUE.STEADY.WAIT",
        "formula": {
            "id": "QUEUE.STABILITY",
            "inputs": [
                {"role": "arrival_rate", "value": 100000, "unit": "count(message)/s"},
                {"role": "service_rate", "value": 90000, "unit": "count(message)/s"},
            ],
            "output_unit": "ratio",
        },
        "result": {"value": 1.1111111111, "unit": "ratio",
                   "interpretation": "finite steady_wait"},
    })
    errors = validate_document(make_document(metric))
    self.assertTrue(any("rho" in error and "steady_wait" in error for error in errors))

def test_unknown_formula_id_is_rejected(self):
    metric = complete_metric({
        "metric_id": "CPU.UNKNOWN",
        "formula": {"id": "CPU.ARBITRARY_EXPRESSION", "inputs": [],
                    "output_unit": "ns"},
        "result": {"value": 1, "unit": "ns"},
    })
    self.assertTrue(any("unknown formula" in error
                        for error in validate_document(make_document(metric))))

def test_result_unit_must_match_formula_output_unit(self):
    metric = complete_metric({
        "metric_id": "CPU.HASH.BAD_UNIT",
        "formula": {
            "id": "RATE.EVENT_WORK",
            "inputs": [
                {"role": "event_rate", "value": 1, "unit": "count(event)/s"},
                {"role": "work_per_event", "value": 2,
                 "unit": "count(hash_probe)/count(event)"},
            ],
            "output_unit": "count(hash_probe)/s",
        },
        "result": {"value": 2, "unit": "ns"},
    })
    self.assertTrue(any("result unit" in error
                        for error in validate_document(make_document(metric))))
```

Use one fixture per behavior only when inline dictionaries would obscure the test.

- [ ] **Step 2: Verify the new tests fail for the intended missing behavior**

Run `python -m unittest discover -s c-perf-analysis/scripts/tests -v`.

Expected: failures naming `RATE.EVENT_WORK`, `QUEUE.STABILITY`, or unit mismatch; existing Task 1 tests remain green.

- [ ] **Step 3: Extend the closed formula catalog minimally**

`RATE.EVENT_WORK` multiplies `event_rate` by typed `work_per_event` and produces that work unit per second. `QUEUE.STABILITY` computes `rho = arrival_rate / service_rate`; a metric requesting finite `steady_wait` is invalid unless `rho < 1` and both rates are `DC`-calibrated.

- [ ] **Step 4: Write `quantitative-analysis-model.md`**

Use this top-level structure:

```markdown
# Quantitative Analysis Model
## Contents
## Core rule
## Scenario ledger
## Semantic units and statistics
## Evidence kind and confidence
## Metric record schema
## Formula catalog
## Static-only output gates
## Mechanical validation
## Common mistakes
## Complete metric example
```

The complete example is `MEM.ADJOUT.REPLICATION`: `1M × 2 × 64–80 B = 128–160 MB`, with both decimal and binary display, allocator/RSS exclusions, and a compile-`sizeof` plus RSS-slope verification hook.

- [ ] **Step 5: Verify reference completeness mechanically**

```bash
rg -n 'scenario_id|SF|SB|EA|DC|C0|C1|C2|C3|MEM.INSTANCE_PRODUCT|RATE.EVENT_WORK|QUEUE.STABILITY|requested|resident|RSS|CPU time|wall-clock|p99' c-perf-analysis/references/quantitative-analysis-model.md
python -m unittest discover -s c-perf-analysis/scripts/tests -v
```

Expected: every term is present and all validator tests pass.

- [ ] **Step 6: Commit the quantitative reference slice**

```bash
git add c-perf-analysis/references/quantitative-analysis-model.md c-perf-analysis/scripts
git commit -m "docs: define quantitative performance model"
```

---

### Task 3: Software Performance Rule Cards

**Files:**
- Create: `c-perf-analysis/references/software-performance-rules.md`
- Create: `c-perf-analysis/demo-case/evaluations/quantitative-scenarios.md`

**Interfaces:**
- Consumes: the scenario, unit, evidence, and formula vocabulary from Task 2.
- Produces: stable rule IDs and a Phase 0 selection index consumed by Tasks 4 and 5.
- Rule-card fields: ID, priority, applicable dimensions, recognition signals, evidence to trace, formula/bounds, common misread, verification hook.

- [ ] **Step 1: Freeze five RED/GREEN behavior scenarios and rubric**

Write cases for:

1. flexible-array entry plus payload percentiles and a demand for exact RSS;
2. fixed 1024-bucket chained hash plus a demand for CPU percentage without calibration;
3. Reactor holding a mutex across `send()` plus a demand to prove no backlog and provide p99;
4. the `656 B` inconsistent total;
5. a mixed serial/parallel pipeline with a flush timer.

Each rubric checks observable behavior: typed formulas, refusal of unsupported precision, missing-input list, evidence/confidence, and verification hook. Preserve the current-skill RED observations separately from the expected GREEN rubric.

- [ ] **Step 2: Write the rule-card reference with a contents table**

Include these P0 cards:

```text
C01 effective build/preprocessing
C02 ABI layout and allocation stride
L01 lifetime and live set
L02 allocation count and allocator layers
D01 container real constants
D02 hidden-loop and fan-out recovery
D03 cache locality and working set
P01 Reactor loop budget
P02 producer-consumer stability
P03 pipeline and copy chain
P04 state-machine amplification
P05 batching/quota/backpressure
P06 timer storm and re-registration
X01 error/timeout/retry feedback
X02 amplified work inside critical sections
```

Include signal-triggered P1 cards for dynamic buffers, multiple indexes, pools, Flyweight/interning, hot-path logging, temporary zero/copy work, and false sharing. Put branch prediction, vectorization, inlining, instruction cache, prefetch, NUMA, and ISA/codegen in P2 with an explicit hotspot prerequisite.

- [ ] **Step 3: Verify every card has all structural fields**

Run a small read-only check that counts card headings and required labels:

```bash
rg -n '^### (C|L|D|P|X)[0-9]{2}' c-perf-analysis/references/software-performance-rules.md
rg -n 'Recognition signals|Evidence to trace|Cost model|Common misread|Verification hook|Dimensions' c-perf-analysis/references/software-performance-rules.md
```

Expected: every card contains all six labels; no card claims exact time or hardware events statically.

- [ ] **Step 4: Commit the rules and evaluation cases**

```bash
git add c-perf-analysis/references/software-performance-rules.md c-perf-analysis/demo-case/evaluations/quantitative-scenarios.md
git commit -m "docs: add C performance analysis rule cards"
```

---

### Task 4: Phase 0 and Phase 3 Integration

**Files:**
- Modify: `c-perf-analysis/SKILL.md:1-185`
- Modify: `c-perf-analysis/references/orchestration-checklist.md:1-120`

**Interfaces:**
- Consumes: formula/reference names from Task 2 and rule IDs from Task 3.
- Produces: checkpoint scenario ledger, selected-rule list, per-agent metrics paths, and Phase 3 validation gate.

- [ ] **Step 1: Run frontmatter and size baseline checks**

```bash
sed -n '1,8p' c-perf-analysis/SKILL.md
wc -l c-perf-analysis/SKILL.md
```

Expected RED observation: description does not start with `Use when`; no quantitative-reference routing exists.

- [ ] **Step 2: Update the frontmatter description**

Use a third-person trigger-only description beginning with:

```yaml
description: Use when statically analyzing performance, capacity, memory, IPC, timers, data structures, locks, event loops, or critical paths in a large C/C++ component and the conclusions need scenario-bound quantitative evidence.
```

Do not summarize the phase workflow in the description.

- [ ] **Step 3: Add progressive-disclosure routing and the core precision gate**

`SKILL.md` must directly link:

- `references/quantitative-analysis-model.md` whenever producing or reconciling numbers;
- `references/software-performance-rules.md` during Phase 0 rule selection and dimension analysis.

State the positive output contract: without calibrated coefficients, output counts, typed work-unit vectors, symbolic formulas, ranges, or conditional bounds.

- [ ] **Step 4: Extend Phase 0 checkpoint fields**

Add build/ABI/compiler/allocator facts, scenario IDs, workload modes, feature states, selected rule IDs, and one `agent_<X>_metrics.json` output path per dimension.

- [ ] **Step 5: Add Phase 3 validation before reconciliation**

Run:

```bash
python c-perf-analysis/scripts/validate_metrics.py <output-dir>/agent_*_metrics.json
```

Arithmetic/unit/schema failures return to the producing agent. Only scope, inclusion, or legitimate assumption differences proceed to range reconciliation.

- [ ] **Step 6: Mirror gates in the orchestration checklist**

Add checkboxes for scenario freeze, rule selection, independent metric files, validator pass, and invalid-arithmetic repair.

- [ ] **Step 7: Verify entrypoint size and routing**

```bash
wc -l c-perf-analysis/SKILL.md
rg -n 'quantitative-analysis-model|software-performance-rules|scenario_id|validate_metrics|agent_<X>_metrics' c-perf-analysis/SKILL.md c-perf-analysis/references/orchestration-checklist.md
```

Expected: `SKILL.md` remains under 500 lines and every routing term is present.

- [ ] **Step 8: Commit orchestration integration**

```bash
git add c-perf-analysis/SKILL.md c-perf-analysis/references/orchestration-checklist.md
git commit -m "feat: gate C performance metrics by scenario"
```

---

### Task 5: A–E Prompt Contract Integration

**Files:**
- Modify: `c-perf-analysis/references/agent-prompt-templates.md:1-181`

**Interfaces:**
- Consumes: `{SCENARIO_FACTS}`, `{RULE_CARDS}`, `{METRICS_FILE}`, metric schema, and formula IDs.
- Produces: human-readable `agent_<letter>_<topic>.md` plus machine-readable `agent_<letter>_metrics.json`.

- [ ] **Step 1: Record current prompt skeleton as the RED baseline**

Verify it lacks `{SCENARIO_FACTS}`, `{RULE_CARDS}`, and `{METRICS_FILE}`:

```bash
rg -n 'SCENARIO_FACTS|RULE_CARDS|METRICS_FILE' c-perf-analysis/references/agent-prompt-templates.md
```

Expected: no matches.

- [ ] **Step 2: Extend the shared skeleton in fixed order**

After `{CHECKPOINT_FACTS}`, add scenario facts, selected rule cards, quantitative output gate, and metrics file. Require metric records only for High-severity numeric claims and values promoted to the final quantitative profile.

- [ ] **Step 3: Add dimension-specific allowed outputs**

Add these gates:

| Dimension | Static-only output |
|---|---|
| A | expiry/s, visit/s, storm count/window, callback work units |
| B | typed operation counts, probe/comparison bounds, exact/conditional layout bytes |
| C | requested bytes, object/allocation/copy counts, logical/copied/wire bytes |
| D | acquisitions/event, critical-section work units, queue bounds, serialized topology |
| E | stage DAG, multiplicities, work vectors, configured gating bounds |

Each template lists the extra calibration required before time, utilization, RSS, wait, throughput, or percentile claims are permitted.

- [ ] **Step 4: Add metric-record output sections to A–E**

Each task block states which formula families it normally uses and requires `includes`, `excludes`, assumption, evidence/confidence, and verification hook fields.

- [ ] **Step 5: Update the pre-dispatch checklist**

The skeleton check now includes ten elements: the original seven plus scenario block, selected rule cards, and metrics output contract.

- [ ] **Step 6: Verify placeholders and output paths**

```bash
rg -n 'SCENARIO_FACTS|RULE_CARDS|METRICS_FILE|Static-only|calibrat|includes|excludes' c-perf-analysis/references/agent-prompt-templates.md
```

Expected: shared placeholders and every A–E calibration gate are discoverable.

- [ ] **Step 7: Commit prompt integration**

```bash
git add c-perf-analysis/references/agent-prompt-templates.md
git commit -m "docs: require quantitative A-E agent outputs"
```

---

### Task 6: Reporting, Terminology, and Demo Repair

**Files:**
- Modify: `c-perf-analysis/references/reporting-standards.md:1-111`
- Modify: `c-perf-analysis/references/glossary.md:1-39`
- Modify: `c-perf-analysis/demo-case/reports/demo_1_pipeline_teardown.md:1-112`
- Modify: `c-perf-analysis/demo-case/reports/demo_2_metric_reconciliation.md:1-102`
- Create: `c-perf-analysis/demo-case/reports/demo_metrics.json`

**Interfaces:**
- Consumes: validated metric records from Tasks 1–5.
- Produces: reader-facing quantitative profile and corrected demo records.

- [ ] **Step 1: Add the current demo metrics as a failing regression fixture**

Create `demo_metrics.json` first with the current inconsistent `656 B` result and run:

```bash
python c-perf-analysis/scripts/validate_metrics.py c-perf-analysis/demo-case/reports/demo_metrics.json
```

Expected: FAIL naming the total metric and recomputed `512 B`.

- [ ] **Step 2: Correct the demo memory components and scope**

Use one consistent floor and ceiling decomposition. Do not hide slack. Represent `adj_out` as `64–80 B` until compiled `sizeof` exists. Ensure every displayed total exactly matches its components.

- [ ] **Step 3: Replace unsupported absolute CPU time in demo 1**

Keep the `1M destinations` multiplicity and emit a typed work-vector or symbolic `1M × c_bestpath` formula. If retaining the `5–15 us` scenario, label it `EA/C1 assumption model`, keep it separate from static findings, and prohibit interpreting it as a measured prediction.

- [ ] **Step 4: Update reporting standards**

Add a scenario/build table, metric ID, evidence kind, confidence, validation status, sensitivity/break-even variables, and semantic separation for CPU/wall, requested/resident/RSS, and logical/copied/wire bytes. State that severity and confidence are independent.

- [ ] **Step 5: Update glossary only for stable terms**

Add bilingual entries for evidence kind, confidence, work-unit vector, requested/usable/resident bytes, scenario, and arithmetic/unit closure. Preserve all existing translation-contract tags.

- [ ] **Step 6: Validate corrected demo and translation vocabulary**

```bash
python c-perf-analysis/scripts/validate_metrics.py c-perf-analysis/demo-case/reports/demo_metrics.json
rg -n 'metric ID|evidence|confidence|scenario|CPU time|wall-clock|requested|resident|logical|copied|wire' c-perf-analysis/references/reporting-standards.md
```

Expected: validator exits `0`; all required report semantics are present.

- [ ] **Step 7: Commit reporting and demo corrections**

```bash
git add c-perf-analysis/references/reporting-standards.md c-perf-analysis/references/glossary.md c-perf-analysis/demo-case/reports
git commit -m "docs: reconcile quantitative performance reports"
```

---

### Task 7: Behavioral GREEN Evaluation and Packaging

**Files:**
- Modify: `c-perf-analysis/README.md:1-45`
- Modify: `c-perf-analysis/demo-case/README.md:1-68`
- Modify only if failures demand it: `c-perf-analysis/SKILL.md`, `c-perf-analysis/references/quantitative-analysis-model.md`, `c-perf-analysis/references/software-performance-rules.md`, `c-perf-analysis/references/agent-prompt-templates.md`

**Interfaces:**
- Consumes: the five frozen evaluation scenarios from Task 3.
- Produces: recorded GREEN outcomes, concise packaging documentation, and the final verified skill.

- [ ] **Step 1: Run five fresh-context GREEN evaluations**

For each scenario, give a fresh agent only the updated skill path and the scenario. Score:

```text
scenario binding
typed formula/work units
unsupported-precision refusal
evidence kind and confidence
includes/excludes
verification hook
no arithmetic or unit error
```

Expected: all mandatory rubric items pass. Record verbatim failures and rationalizations.

- [ ] **Step 2: Run wording micro-tests against the no-new-guidance control**

Use five fresh samples for the strongest behavior-shaping rule: a user demands an exact number despite missing calibration. Compare the current-skill control with the updated positive output contract. Manually read every result; do not score template echoes as compliance.

- [ ] **Step 3: Refactor only observed failures**

If an agent still invents precision or omits a required field, strengthen the structural output slot or observable conditional that failed. Do not add generic C-performance prose. Rerun the failed scenario until green.

- [ ] **Step 4: Update README files**

Document the quantitative layer, rule-card selection, per-agent JSON metrics, validator command, and corrected demo purpose. Keep README descriptive; runtime rules remain in `SKILL.md` and references.

- [ ] **Step 5: Run full mechanical validation**

```bash
python -m unittest discover -s c-perf-analysis/scripts/tests -v
python c-perf-analysis/scripts/validate_metrics.py c-perf-analysis/scripts/tests/fixtures/valid-memory.json c-perf-analysis/demo-case/reports/demo_metrics.json
python /Users/Alick/.codex/skills/.system/skill-creator/scripts/quick_validate.py c-perf-analysis
git diff --check
wc -l c-perf-analysis/SKILL.md
```

Expected: all unit tests pass, metric files validate, skill quick-validation passes, no whitespace errors, and `SKILL.md` is under 500 lines.

- [ ] **Step 6: Review against the design spec**

Check every Acceptance Criteria bullet in `docs/superpowers/specs/2026-09-01-c-perf-analysis-quantitative-model-design.md` and record any gap before claiming completion.

- [ ] **Step 7: Commit packaging and final refinements**

```bash
git add c-perf-analysis
git commit -m "feat: add quantitative C performance analysis"
```

Do not push or open a pull request unless the user explicitly asks.

---

## Execution Schedule

1. Task 1 is serial because every other task depends on the metric contract.
2. After Task 1, Tasks 2 and 3 may run in parallel; they own disjoint files.
3. After Tasks 2 and 3, Tasks 4 and 5 may run in parallel; they own disjoint files and consume frozen interfaces.
4. Task 6 runs after validator, references, and prompt contracts are stable.
5. Task 7 is serial integration, behavioral evaluation, and final verification.

## Checkpoints

- After Task 1: validator RED/GREEN cycle passes and catches the current `656 B` error.
- After Tasks 2–3: formula IDs, scenario schema, evidence vocabulary, and rule IDs are frozen.
- After Tasks 4–5: Phase 0/1/3 orchestration and all A–E prompts use the same contract.
- After Task 6: demo records are mechanically valid and reader-facing semantics agree.
- After Task 7: behavioral GREEN evaluation, quick validation, full tests, and spec coverage all pass.
