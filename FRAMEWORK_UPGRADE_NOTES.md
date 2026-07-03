# MIPLIB-NL Evaluation Framework Upgrade Notes

Date: 2026-07-03

This note summarizes the framework upgrades made today to improve the MIPLIB-NL evaluation harness beyond the original one-shot code-generation baseline.

## Background

The original evaluation flow was simple and clear:

```text
build prompt -> call model -> extract Python code -> execute code -> compare objective value
```

That is a good Pass@1 baseline, but it leaves performance on the table for NL-to-optimization tasks. Many failures in this benchmark are not purely mathematical; they often come from solver API mistakes, CSV schema misunderstanding, non-contiguous IDs, missing output formatting, or small runtime errors.

The upgrade keeps the baseline behavior available by default, while adding optional stronger modes inspired by current agent/evaluation patterns: stateful repair loops, error-specific feedback, candidate generation, and data profiling.

## Main Changes

### 1. Error-Driven Repair Loop

Added an optional repair loop in `src/run_one.py`.

When generated code fails, the framework can now:

1. Execute the generated code.
2. Evaluate stdout against the reference objective.
3. Classify the failure using `src/error_classifier.py`.
4. Build a repair prompt containing:
   - original task prompt
   - current code
   - stdout/stderr
   - solver status
   - objective/gap information
   - compact history of previous attempts
   - error-specific repair hints
5. Ask the same model to return a full repaired script.
6. Re-execute and keep the better candidate.

This is controlled by:

```yaml
max_repair_rounds: 0
```

Default is `0`, so old baseline-style runs are preserved unless explicitly enabled.

Example:

```powershell
.\batch_run.ps1 -Limit 2 -MaxRepairRounds 1 -NoResume
```

### 2. Repair Policy Module

Added `src/repair_policy.py`.

This module centralizes logic for:

- error-specific repair hints
- compact attempt summaries
- candidate scoring
- deciding whether a repaired candidate is better than the current one
- candidate generation strategies for Best-of-N runs

The scoring priority is:

```text
pass > executable > objective_found > smaller relative_gap > shorter runtime
```

This makes the framework less dependent on a single brittle retry and more like a controlled stateful workflow.

### 3. Deterministic Code Sanitizer Integration

The existing `src/generated_code_fixer.py` is now connected to the main run path.

Generated and repaired code is passed through deterministic sanitizer rules before execution. Applied fixes are recorded in the result JSONL fields:

- `initial_sanitizer_fixes`
- `sanitizer_fixes`

This helps fix known mechanical issues such as SciPy MILP import/signature mistakes without asking the model again.

### 4. Best-of-N Candidate Generation

Added optional multi-candidate generation.

Instead of producing only one solution, the framework can generate multiple independent candidates, execute each one, optionally repair each one, and select the best result automatically.

Controlled by:

```yaml
num_candidates: 1
```

Default is `1`, preserving old behavior.

Recommended stronger setting:

```powershell
.\batch_run.ps1 -Limit 2 -NumCandidates 3 -MaxRepairRounds 1 -NoResume
```

Artifacts:

- candidate 1 keeps historical artifact names
- candidate 2+ use `_candidate<N>` suffixes
- repair artifacts use `_repair<N>` suffixes

Result records now include:

- `num_candidates`
- `selected_candidate_id`
- `candidate_summaries`

### 5. CSV Schema Profiler

Added `src/schema_profiler.py`.

The prompt now includes a deterministic CSV data profile in addition to raw schemas and samples. The profile includes:

- row counts
- inferred column type hints
- empty/non-empty counts
- unique value counts
- numeric min/max ranges
- small categorical sample values
- warnings for non-contiguous integer IDs
- hints to map raw IDs to dense zero-based indices before array/sparse-matrix use

This profile is injected through:

- `src/datasets/miplib_nl.py`
- `src/prompt_builder.py`
- `prompts/nl_schema_to_code.txt`

This is expected to reduce common modeling-code failures around CSV interpretation and indexing.

### 6. Batch Runner and Config Updates

Updated `src/run_batch.py` to pass the new options into `run_eval_case`:

- `max_repair_rounds`
- `num_candidates`

Updated `configs/eval.yaml`:

```yaml
max_repair_rounds: 0
num_candidates: 1
```

Updated `batch_run.ps1` with:

```powershell
-MaxRepairRounds <int>
-NumCandidates <int>
```

The resume key now includes both enhancement parameters. This prevents runs with different protocols from being incorrectly skipped as already completed.

## Recommended Usage

### Baseline-Compatible Run

This preserves the original one-shot protocol except for deterministic sanitizer recording:

```powershell
.\batch_run.ps1 -Limit 2 -NoResume
```

### Repair-Only Enhanced Run

```powershell
.\batch_run.ps1 -Limit 2 -MaxRepairRounds 1 -NoResume
```

### Stronger Best-of-N + Repair Run

```powershell
.\batch_run.ps1 -Limit 2 -NumCandidates 3 -MaxRepairRounds 1 -NoResume
```

This is the recommended smoke-test setting for checking whether the enhanced framework improves model success rates.

## Files Changed

New files:

- `src/repair_policy.py`
- `src/schema_profiler.py`
- `FRAMEWORK_UPGRADE_NOTES.md`

Modified files:

- `src/run_one.py`
- `src/run_batch.py`
- `src/datasets/miplib_nl.py`
- `src/prompt_builder.py`
- `prompts/nl_schema_to_code.txt`
- `configs/eval.yaml`
- `batch_run.ps1`
- `README_EVAL.md`

## Notes on Evaluation Fairness

These upgrades change the evaluation protocol when enabled. For fair comparison:

- report `max_repair_rounds`
- report `num_candidates`
- keep solver backend fixed
- keep timeout fixed
- keep dataset version fixed
- do not compare one-shot Pass@1 directly against Best-of-N or repaired runs without labeling them separately

Suggested labels:

```text
Pass@1 baseline: num_candidates=1, max_repair_rounds=0
Repair@1:        num_candidates=1, max_repair_rounds=1
BestOf3+Repair:  num_candidates=3, max_repair_rounds=1
```

## Verification Status

The code changes were reviewed at the text/diff level in the current workspace. Python execution was not completed from this Codex shell because the shell PATH did not expose a usable Python interpreter.

Recommended local checks:

```powershell
python -m compileall src
python -m src.run_batch --config configs/eval.yaml --models zhipu_glm47_flash --datasets miplib_nl --limit 1 --dry-run --num-candidates 3 --max-repair-rounds 1
```

