# Auto-Refine: Prompt Improvement Pipeline

Converts the existing Python code modification loop into an **evaluation prompt (checklist) improvement loop**.

## Overview

```
For the evaluation prompt:
  golden set test → failure mode detection → prompt improvement → repeat
```

### Previous (AS-IS)

- **Target**: extract_claims.py, verify_coherence.py (Python code)
- **Tools**: Python + 27 GitHub repos
- **Failure modes**: regex mismatches, parsing errors, edge case omissions
- **Fix method**: Direct Python code modification

### New (TO-BE)

- **Target**: `agents/native-evaluator.md`, `references/eval-rubric.md`, `references/claim-taxonomy.md`
- **Tools**: Claude Code native (Read, Glob, Grep, Agent)
- **Failure modes**: Non-determinism, false positives/negatives, classification errors
- **Fix method**: Prompt text improvement (clarifying judgment criteria, adding few-shot examples)

---

## Execution Process

### Step 1: Prepare Golden Set

Use test scenarios defined in `evals/evals.json`. Each scenario includes:

- Inline CLAUDE.md text or actual GitHub repo reference
- Project fixtures (package.json, directory structure)
- Expected results (expected claims, statuses, scores)

### Step 2: Run Evaluation

Run the native-evaluator agent for each golden set item:

```
Agent(native-evaluator):
  Inputs:
    - claude_md: eval_set.claude_md
    - project_fixtures: eval_set.project_fixtures
    - eval_rubric: references/eval-rubric.md
    - claim_taxonomy: references/claim-taxonomy.md
    - output_schema: references/output-schema.json
  Output: evaluation_result.json
```

### Step 3: Compare Results

Compare evaluation results against expected values:

```
For each eval_set:
  1. Extracted claim list vs expected claim list
     - Missing claims (should exist but wasn't extracted)
     - Over-extraction (shouldn't exist but was extracted)
  2. Each claim's status vs expected_status
     - False positive: judged as verified when actually stale
     - False negative: judged as stale when actually verified
  3. Whether must_not_extract items were actually extracted
  4. Whether conflicts were detected
  5. Score range appropriateness
```

### Step 4: Failure Mode Analysis

| Failure Mode | Symptom | Prompt Improvement Direction |
|---|---|---|
| **Extracting claims from code blocks** | must_not_extract items extracted as claims | Strengthen "distinguishing code block content" rule in native-evaluator.md |
| **Misidentifying comparative context** | "like React" extracted as framework claim | Add comparative context examples to claim-taxonomy.md |
| **Not filtering built-in commands** | npm install extracted as command claim | Expand built-in exclusion list in claim-taxonomy.md |
| **False positive (stale→verified)** | Non-existent path judged as verified | Tighten verification procedures in eval-rubric.md |
| **False negative (verified→stale)** | Existing command judged as stale | Expand verification sources in eval-rubric.md (scripts, targets, etc.) |
| **Unidentified vague/generic** | Ambiguous instruction classified as convention | Strengthen vague/generic identification criteria in claim-taxonomy.md |
| **Undetected conflicts** | Contradictory instructions not detected | Add examples to conflict detection procedure in eval-rubric.md |
| **Non-determinism** | Different results for same input (N=3 mismatch) | Change judgment criteria to clearer binary conditions |

### Step 5: Modify Prompts

Modify target files based on failure modes:

| Target File | Modification Content |
|---|---|
| `agents/native-evaluator.md` | Evaluation process, tool call strategy, critical rules |
| `references/eval-rubric.md` | Judgment criteria, verification methods, scoring rules |
| `references/claim-taxonomy.md` | Claim type definitions, examples, edge cases |

Modification methods:
- **Clarify judgment criteria**: "may be ~" → "if ~, then verified; otherwise stale"
- **Add few-shot examples**: Add failed cases as examples
- **Specify edge cases**: Explicitly define ambiguous situations discovered in failure modes

### Step 6: Re-evaluate and Repeat

Repeat Steps 2-5 with the modified prompts.

**Termination conditions**:
- All golden set items match expected results
- When running N=3, all claim statuses agree at least 2/3 of the time
- Scores are within expected range

---

## How to Run

auto-refine is run directly within a Claude Code session:

```
/claude-md-creator:auto-refine
```

When executed, it performs the following:

1. Load `evals/evals.json`
2. Run native_evaluator agent for each eval_set
3. Compare results against expected values
4. Output failure mode analysis report
5. Suggest prompt modifications (or auto-apply)
6. Re-run after modifications

### Progress Tracking

Record results of each iteration:

```
results/
├── iter-1.json    # Iteration 1 results
├── iter-2.json    # Iteration 2 results
└── summary.json   # Overall progress summary
```

---

## Differences from Previous Python Harness

| Item | AS-IS (Python) | TO-BE (Native) |
|---|---|---|
| Refine target | Python code (regex, parsers) | Prompt text (judgment criteria, examples) |
| Test tools | Python + subprocess | Claude Agent tools |
| Test environment | Local clone of GitHub repos | Inline fixtures or local repos |
| Result comparison | Python assertion | Claude compares against expected values |
| Fix method | Python AST/regex modification | Markdown prompt editing |
| Reproducibility | Deterministic (same input = same output) | Non-deterministic (mitigated by N=3 majority vote) |
| Cross-platform | Requires Python | No dependencies |
