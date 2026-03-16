---
name: auto-refine
description: |
  An automated pipeline that tests and improves the claude-md-creator native evaluation system's prompts.
  Runs native_evaluator against a golden set (evals.json), compares results with expected values to detect
  failure modes, then modifies the prompts. If a numeric argument is provided, it automatically runs
  N iterations via Ralph Loop.
  Triggers: "/auto-refine", "eval test", "prompt improvement", "golden set test",
  "native eval test", "evaluation accuracy test"
---

# Auto-Refine — Automated Native Evaluation Prompt Improvement

An automation pipeline that iterates: golden set test -> failure mode detection -> prompt improvement.

## Usage

```
# Single run (1 iteration)
/claude-md-creator:auto-refine

# N iterations (Ralph Loop auto-enabled, early exit when all tests pass)
/claude-md-creator:auto-refine 10
```

---

## Execution Process

**Print the following banner first upon execution:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔧 Skill Activated: auto-refine
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Step 0: Activate Ralph Loop (when argument is a number)

Extract a number from the argument. If a number is found, automatically activate Ralph Loop:

```bash
mkdir -p .claude
cat > .claude/ralph-loop.local.md <<'STATEEOF'
---
active: true
iteration: 1
max_iterations: {N}
completion_promise: "ALL_TESTS_PASS"
started_at: "{ISO_TIMESTAMP}"
---

/claude-md-creator:auto-refine
STATEEOF
```

Then print the following message:

```
🔄 Ralph Loop activated — max {N} iterations, auto-terminates when all tests pass
```

If no number is found, proceed with a single run.

### Step 1: Check Progress State

Check whether results from a previous iteration exist:

```
Read: results/summary.json (if exists)
```

If found, read the previous iteration's pass/fail status and modification history, then determine which scenarios to focus on in this iteration.

If not found, treat as a first run and target all scenarios.

### Step 2: Load Golden Set

```
Read: evals/evals.json
```

Prepare each scenario in the `eval_sets` array as a test target.

### Step 3: Load Reference Documents

Read all prompt files used for evaluation:

```
Read: agents/native-evaluator.md
Read: references/eval-rubric.md
Read: references/claim-taxonomy.md
Read: references/output-schema.json
```

### Step 4: Run Evaluation Per Scenario

For each eval_set, use an Agent to perform the evaluation. The agent receives only the evaluation prompts (native-evaluator.md, eval-rubric.md, claim-taxonomy.md) and the scenario's CLAUDE.md + project_fixtures. **Expected values are NOT provided** — this is a blind test.

For efficiency, batch 2-4 scenarios together and run them as parallel agents.

#### Agent Prompt Structure

```
You are a CLAUDE.md evaluator. Follow the evaluation rules EXACTLY.
[native-evaluator.md content]
[eval-rubric.md content]
[claim-taxonomy.md content]

--- Scenario ---
CLAUDE.md: [inline text]
Project Fixtures: [package.json, directory_structure, etc.]

--- Output ---
JSON: { scenario_id, claims: [{id, type, raw_text, status}], conflicts: [{a, b}] }
```

### Step 5: Compare Results

Compare actual results against expected values (`expected`) for each scenario:

```
For each scenario:
  ✅ PASS conditions:
    - Extracted claim count matches expected claim count (±1 tolerance)
    - Each claim's status matches expected_status
    - must_not_extract items were not extracted
    - must_not_extract_as_framework items were not classified as framework
    - must_not_extract_as_command items were not classified as command
    - At least conflicts_min conflicts were detected
    - Score is within expected range
  ❌ FAIL conditions:
    - Any of the above conditions not met
```

### Step 6: Results Report

Generate user-facing output (reports, summaries, status messages) in the same language as the CLAUDE.md being evaluated. If no CLAUDE.md context exists, follow the user's system language.

Output the test results as a table:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Auto-Refine Results (iter {N})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| # | Scenario              | Result | Details                          |
|---|----------------------|--------|----------------------------------|
| 1 | minimal-valid        | ✅     |                                  |
| 2 | stale-paths          | ❌     | Misjudged path-1 as verified     |
| 3 | vague-generic        | ✅     |                                  |
| ...                                                                    |

Passed: 6/8 (75%)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Step 7: Failure Analysis and Prompt Modification

When there are failed scenarios:

1. **Classify failure modes**:
   - `false_positive`: Judged as verified when actually stale
   - `false_negative`: Judged as stale when actually verified
   - `wrong_type`: Misclassified claim type
   - `over_extraction`: Extracted something that should not have been extracted
   - `under_extraction`: Failed to extract something that should have been extracted
   - `missed_conflict`: Failed to detect a conflict
   - `score_out_of_range`: Score outside expected range

2. **Modify prompts** (target file determined by failure mode):

   | Failure Mode | Modification Target |
   |---|---|
   | false_positive / false_negative | Strengthen verification procedure in `references/eval-rubric.md` |
   | wrong_type | Reinforce type classification criteria in `references/claim-taxonomy.md` |
   | over_extraction / under_extraction | Claim identification rules in `agents/native-evaluator.md` |
   | missed_conflict | Conflict detection rules in `references/eval-rubric.md` |
   | score_out_of_range | Review score calculation logic |

3. **Apply modifications**: Use the Edit tool to modify relevant sections of the prompt file
   - Make classification criteria clearer (ambiguous expressions -> binary conditions)
   - Add failure cases as few-shot examples
   - Explicitly define edge cases

### Step 8: Save Results

```
Write: results/summary.json
```

```json
{
  "last_iteration": 3,
  "timestamp": "2026-03-15T13:30:00+09:00",
  "results": {
    "total": 8,
    "passed": 6,
    "failed": 2,
    "pass_rate": 75
  },
  "failed_scenarios": [
    {
      "id": "stale-paths",
      "failure_mode": "false_positive",
      "detail": "Misjudged path-1 as verified",
      "fix_applied": "Added directory_structure cross-reference step to path verification in eval-rubric.md"
    }
  ],
  "prompt_changes": [
    {
      "file": "references/eval-rubric.md",
      "change": "Added exact match condition to path_reference verification",
      "iteration": 3
    }
  ],
  "history": [
    { "iteration": 1, "passed": 4, "total": 8 },
    { "iteration": 2, "passed": 6, "total": 8 },
    { "iteration": 3, "passed": 6, "total": 8 }
  ]
}
```

### Step 9: Termination Decision

```
if all scenarios pass (pass_rate == 100):
  git add + commit modified prompt files
  output: "✅ All golden set tests passed!"
  output: <promise>ALL_TESTS_PASS</promise>
else:
  output: "🔄 {pass_rate}% passed — continuing in next iteration"
  (Ralph Loop provides the same prompt again to start the next iteration)
```

---

## Iteration Flow Example

```
/claude-md-creator:auto-refine 10

🔄 Ralph Loop activated — max 10 iterations

Iteration 1: Run evaluation → 4/8 passed → Analyze 4 failures → Modify prompts → Commit
Iteration 2: Run evaluation → 6/8 passed → Analyze 2 failures → Modify prompts → Commit
Iteration 3: Run evaluation → 7/8 passed → Analyze 1 failure → Modify prompts → Commit
Iteration 4: Run evaluation → 8/8 passed → <promise>ALL_TESTS_PASS</promise> → Loop terminated
```

### Commit Message Format

When prompts are modified in each iteration, auto-commit:

```
chore: auto-refine prompt improvement (iter {N}, {pass_rate}%)

- {summary of modifications}
```

When all tests pass:

```
chore: auto-refine complete — all golden set tests passed (iter {N})
```

---

## Important Rules

1. **Modify prompts only**: Do not modify expected values or test scenarios in evals.json. Improve prompts to match expected values.
2. **One at a time**: Even if multiple failure modes are found in one iteration, fix the most impactful one first.
3. **Record modifications**: All modifications are recorded in summary.json for reference in the next iteration.
4. **Prevent infinite loops**: If the same modification is repeated more than twice, change the approach or report to the user.
5. **Commit granularity**: Each iteration's prompt modifications as a single commit.
6. **Trust expected values**: The expected values in evals.json are ground truth. If evaluation results differ, the prompt is wrong.
