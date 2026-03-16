# SPEC: Native Eval — Script Removal and Transition to Claude Native Evaluation

## 1. Overview

This spec transitions Phases 1-2 of the current CLAUDE.md evaluation pipeline from Python scripts (extract_claims.py 861 lines + verify_coherence.py 766 lines) to **Claude native tools (Read, Glob, Grep) + checklist prompts**. All evaluation is performed using only Claude Code built-in tools, without any external scripts (Python, Bash, Node.js).

### Core Motivation

CLAUDE.md is a file that Claude **reads as plain text and understands in natural language**. However, the current evaluation system tries to "understand" this natural language using Python regular expressions. This paradigm mismatch resulted in 7 auto-refine iterations and 1,627 lines of edge-case code.

Since Claude is already running when the skill is invoked, it can simply read the CLAUDE.md directly and verify against the codebase to perform the evaluation.

### Goals

- Delete all Python/Bash/Node.js scripts → replace with agent prompts (checklists)
- Cross-platform (macOS, Linux, Windows) native operation — no external runtime dependencies
- Evaluation results as JSON structured output
- Redesign the entire pipeline (5-Phase)

---

## 2. Background

### Current Architecture (AS-IS)

```
CLAUDE.md
    │
    ▼
Phase 1: extract_claims.py (Python regex, 861 lines)
    │   - Backtick token regex extraction
    │   - 8-type classification (command, path, framework, convention, naming, env, vague, generic)
    │   - Code block state machine, comparison context exclusion, built-in command filtering, etc. edge cases
    ▼
claims.json
    │
    ▼
Phase 2: verify_coherence.py (Python parsing, 766 lines)
    │   - 6 dependency parsers for package.json, pyproject.toml, Cargo.toml, etc.
    │   - Monorepo sub-package scanning
    │   - os.path.exists(), find command, tsconfig.json alias resolution
    ▼
coherence_report.json
    │
    ▼
Phase 3: generate_eval_tasks.py + eval_generator agent
Phase 4: blind_coder + blind_judge agent (A/B test)
Phase 5: aggregate_results.py + generate_report.py → HTML report
```

### Problems

| Problem | Cause |
|------|------|
| Tracking ``` vs ~~~ inside code blocks | Regex has no awareness of context |
| Excluding `"like React"` comparison context | Regex has no understanding of meaning |
| 30+ built-in command exclusion list | Regex cannot determine "user-defined scripts" |
| Parsing PEP 735, Poetry, PEP 621 separately | Python must directly understand every dependency format |
| Monorepo sub-package depth limit + timeout | Deterministic control of filesystem traversal needed |
| 7 auto-refine iterations | Time spent trying to make Python understand natural language |
| No cross-platform support | Python/Bash runtime may not exist on Windows |

### Core Insight

**Claude can already understand all of this in natural language.** When the skill is invoked and Claude reads the CLAUDE.md via Read, it naturally distinguishes whether something is an example inside a code block or an actual instruction, whether it is a comparison context or an actual framework reference. No separate regex or state machine is needed.

### Evaluation Criteria Based on Official Documentation (https://code.claude.com/docs/en/memory)

| Item | Official Recommendation | Evaluation Method |
|------|----------|----------|
| **Size** | 200 lines or fewer | Claude checks line count after Read |
| **Structure** | Grouped with markdown headers and bullets | Claude determines presence of structure |
| **Specificity** | Specific enough to be verifiable | Claude evaluates specificity of each instruction |
| **Consistency** | No contradictory instructions | Claude detects conflicts between instructions |
| **Include** | Build/test commands, coding style, architecture patterns, workflows | Claude verifies existence of required sections |
| **Exclude** | Things inferable from code, git history, long references, generic advice | Claude identifies such patterns |
| **@import** | Relative/absolute paths, max 5 depth | Claude verifies path validity |
| **.claude/rules/** | Conditional loading via paths frontmatter | Claude verifies frontmatter validity |

---

## 3. Detailed Requirements

### Functional Requirements

1. **Remove all scripts**: Evaluate using only Claude built-in tools, without Python, Bash, or Node.js scripts
2. **2-step evaluation structure**:
   - Step 1 (Quantitative Check): Structural/quantitative evaluation based on official documentation criteria
   - Step 2 (Semantic Check): Convert CLAUDE.md into semantic-unit checklist → verify against codebase
3. **Maintain existing 8 claim types**: command, path_reference, framework_reference, convention, naming_pattern, env_reference, vague, generic
4. **JSON structured output**: Produce evaluation results as JSON
5. **Scoring**: Produce coherence_score + grade (A-F)
6. **Cross-platform**: Works natively on macOS, Linux, and Windows

### Non-functional Requirements

1. **Non-determinism mitigation**: Ensure consistency via N=3 majority vote
2. **Time**: Complete within ~1-2 minutes for Quick Eval
3. **No external dependencies**: Works anywhere Claude Code is installed

---

## 4. Technical Design

### New Architecture (TO-BE)

```
CLAUDE.md
    │
    ▼
Step 1: Quantitative Check (Claude reads directly via Read)
    │   - Total line count (200 lines or fewer?)
    │   - Presence of markdown header/bullet structure
    │   - Presence of backtick commands
    │   - @import path validity
    │   - .claude/rules/ frontmatter validity
    │   - Presence of essential sections (build, test, code style)
    │
    ▼
Step 2: Semantic Check (Claude evaluates in natural language)
    │   - Identify claims by semantic units from each CLAUDE.md instruction
    │   - Verify each claim against the codebase (Read/Glob/Grep)
    │   - Determine verdict: verified / stale / partial / unverifiable / vague / generic
    │   - Identify ambiguous instructions, generic advice, redundancy inferable from code
    │   - Detect contradictions/conflicts between instructions
    │
    ▼
evaluation_result.json (structured output)
```

### Step 1: Quantitative Check — Evaluation Criteria

Mechanical check items derived from the official documentation (https://code.claude.com/docs/en/memory):

```
1. Size: Is it 200 lines or fewer?
2. Structure: Are there markdown headers (#)? Are bullets (-, *) used?
3. Commands: Are executable commands included within backticks?
4. @import: If @path syntax exists, does the target file exist? (within 5 depth)
5. .claude/rules/: If paths frontmatter exists, are the glob patterns valid?
6. Essential sections: Does at least one of build/test/code style exist?
```

### Step 2: Semantic Check — Evaluation Criteria

Claude reads the CLAUDE.md, converts each instruction into a checklist item, then verifies:

```
For each instruction:

1. Commands: Verify if actually executable
   → Check existence in package.json scripts, Makefile targets, Cargo.toml, etc.

2. Paths: Verify if actually exist
   → Confirm file/directory existence via Glob/Read

3. Frameworks: Verify if present in dependencies
   → Check in dependency files

4. Conventions: Evaluate if specific and verifiable
   → Sample code to confirm actual adherence

5. Ambiguity: Flag vague/generic instructions
   → Identify ambiguous instructions like "Write clean code"
```

### JSON Output Schema

```json
{
  "source_file": "CLAUDE.md",
  "quantitative_check": {
    "line_count": 142,
    "over_200_lines": false,
    "has_headers": true,
    "has_bullets": true,
    "has_commands": true,
    "imports_valid": true,
    "rules_frontmatter_valid": true,
    "has_essential_sections": true,
    "score": 100
  },
  "semantic_check": {
    "coherence_score": 82,
    "grade": "B",
    "summary": {
      "verified": 12,
      "stale": 3,
      "partial": 1,
      "unverifiable": 2,
      "vague": 2,
      "generic": 1
    },
    "claims": [
      {
        "id": "cmd-1",
        "type": "command",
        "raw_text": "`pnpm dev` — start dev server",
        "source_section": "Build & Run",
        "status": "verified",
        "evidence": "Found in package.json scripts.dev",
        "confidence": 1.0
      },
      {
        "id": "path-3",
        "type": "path_reference",
        "raw_text": "API handlers live in `src/api/handlers/`",
        "status": "stale",
        "evidence": "Directory does not exist. Found `src/app/api/` instead",
        "confidence": 0.0,
        "fix_suggestion": "Update path to `src/app/api/`"
      },
      {
        "id": "vague-1",
        "type": "vague",
        "raw_text": "Write clean, maintainable code",
        "status": "vague",
        "evidence": "Too vague to verify — violates specificity guideline",
        "fix_suggestion": "Replace with specific rules like 'Use named exports' or 'Max function length 50 lines'"
      }
    ],
    "conflicts": [
      {
        "claim_a": "conv-2",
        "claim_b": "conv-5",
        "description": "'Use default exports' conflicts with 'Use named exports only'"
      }
    ]
  },
  "overall_score": 85,
  "overall_grade": "B",
  "top_issues": [
    {
      "severity": "HIGH",
      "claim_id": "path-3",
      "summary": "Stale path reference: src/api/handlers/ → src/app/api/"
    }
  ],
  "recommendations": [
    "Update 3 stale path references",
    "Remove or concretize 2 vague instructions",
    "Add missing test command (no test script found)"
  ]
}
```

### Non-determinism Mitigation: N=3 Majority Vote

```
Run 3 independent evaluations with the same CLAUDE.md + same checklist prompt
    │
    ▼
For each claim, adopt the verdict that matches in 2 or more out of 3 runs
    │
    ▼
Produce consensus_result.json
```

#### Majority Vote Logic

- Each evaluation round runs as an independent agent session
- For each claim's `status` value, the value that matches in 2 or more out of 3 runs is adopted as the final status
- If any round has a score that differs by more than ±5%, that round is excluded as an outlier
- If all 3 rounds produce different results, the most conservative (lowest score) is adopted

### Auto-refine Transition

#### Current (AS-IS)

```
For Python code:
  Test on 27 repos → detect failure modes → fix Python code → repeat
```

#### New (TO-BE)

```
For evaluation prompts:
  Test on N repos → review consistency/accuracy → improve prompts (checklists) → repeat
```

- **Refine target**: Evaluation criteria prompt (checklist, rubric, verdict criteria)
- **Failure modes**:
  - Results differ significantly across 3 runs for the same CLAUDE.md (non-determinism)
  - Something clearly verified is judged as stale (false negative)
  - Something clearly stale is judged as verified (false positive)
  - Missed identification of vague/generic
  - Undetected contradictions/conflicts between instructions
- **Improvement method**: Add clearer verdict criteria to the prompt, or augment with examples (few-shot)

### Phase 3-5 Direction

Phases 3-5 (eval task generation, blind A/B test, report) will be reviewed separately after the Phase 1-2 transition is complete. Since they are already agent-based, they are expected to be compatible once the claims.json input format is aligned with the new JSON schema.

---

## 5. Edge Cases

### Automatically Resolved by Script Removal

| Edge Case | Reason for Resolution |
|---|---|
| Backtick tracking inside code blocks | Claude naturally distinguishes by context |
| `"like React"` comparison context | Claude understands meaning |
| Built-in commands vs user scripts | Claude can distinguish |
| Parsing various dependency file formats | Claude reads via Read and understands the content |
| @-import pointer file interpretation | Claude inspects the file content and makes a judgment |
| tsconfig.json alias resolution | Claude reads tsconfig and resolves paths |
| Windows/macOS/Linux path differences | Claude's tools (Read, Glob) abstract away the OS |

### New Considerations to Manage

| Edge Case | Mitigation Strategy |
|---|---|
| Non-determinism (same input, different output) | N=3 majority vote |
| Large CLAUDE.md (200+ lines) | Warn as official recommendation violation + evaluate by segments |
| Monorepo (multiple CLAUDE.md files) | Evaluate each CLAUDE.md independently, then aggregate |
| Claude making excessive tool calls | Specify efficient verification strategies in the prompt (e.g., read package.json once and batch-verify all commands) |
| Increased evaluation cost | N=3 x agent sessions. Provide N=1 cost-saving option for Quick Eval |

---

## 6. Trade-off Log

### Decision: Remove All Scripts → Claude Native Only

**Advantages**:
- Delete 1,627 lines + auto-refine harness Python code
- Edge cases automatically resolved (Claude excels at natural language understanding)
- Cross-platform (no Python/Bash runtime required)
- Complete elimination of external dependencies for plugin distribution
- Auto-refine simplified to "prompt improvement"

**Disadvantages**:
- Non-determinism (mitigated by N=3 majority vote)
- Increased evaluation cost (API call costs)
- No deterministic reproducibility (slightly different results for the same input)

### Rejected Alternatives

| Alternative | Reason for Rejection |
|---|---|
| Anthropic SDK (pip install) | Python package dependency for plugin distribution → violates native constraint |
| Hybrid (Python + Claude) | Judging the "deterministic/semantic boundary" is essentially the same problem |
| Claude CLI (`claude -p`) | No need to invoke a separate process from within the plugin — already inside a Claude session |
| Bash script (quantitative check) | Bash not guaranteed on native Windows |
| Python script (quantitative check) | Python not guaranteed on Windows. Cross-platform constraint |
| Node.js script (quantitative check) | Although Claude Code is Node.js-based, direct invocation of the node binary is not guaranteed |
| No scripts, Claude only (quantitative) | **Adopted** — Claude performs quantitative checks directly after Read. Simplest approach with no dependencies |

### Evaluation Philosophy Review

The current evaluation philosophy (coherence 50% + effectiveness 50%) is also subject to review, but will be addressed separately after the Phase 1-2 transition.

---

## 7. Implementation Checklist

### Step 1: Checklist Prompt Design

- [x] Write quantitative check criteria prompt (6 items based on official docs) → `references/eval-rubric.md`
- [x] Write semantic check criteria prompt (claim extraction + verification rubric) → `references/eval-rubric.md`
- [x] Finalize JSON output schema → `references/output-schema.json`
- [x] Specify verdict criteria per claim type (with examples) → `references/claim-taxonomy.md`
- [x] Efficient verification strategy prompt (minimize tool calls) → `agents/native-evaluator.md` Critical Rules

### Step 2: SKILL.md Rewrite

- [x] Rewrite Quick Eval section based on checklist prompts → `SKILL.md`
- [x] Remove all Python script invocations → Complete (branched from main, no scripts)
- [x] Write evaluation agent prompt file → `agents/native-evaluator.md`
- [x] Specify N=3 majority vote execution flow → `SKILL.md` N=3 Consensus Mode

### Step 3: Auto-refine Transition

- [x] Change auto-refine target from Python code → prompts → `references/prompt-refine.md`
- [x] Redefine failure modes (non-determinism, false positive/negative) → `references/prompt-refine.md` Step 4
- [x] Design prompt improvement loop → `references/prompt-refine.md` Step 1-6
- [x] Redesign golden set test approach → `evals/evals.json`

### Step 4: Cleanup

- [x] Delete entire scripts/ directory → Branched from main, scripts not included (residual __pycache__ removed)
- [x] Review/remove auto-refine/harness/ Python scripts → Residual __pycache__ removed
- [x] Update references/claim-taxonomy.md → Newly created
- [x] Update evals/evals.json to match new structure → Newly created (8 test scenarios)

### Step 5: Validation

- [ ] Run new evaluation on 2 existing golden set repos, compare with previous results
- [ ] Verify N=3 majority vote consistency (within ±5% across 3 evaluations of the same CLAUDE.md)
- [ ] Verify cross-platform operation (minimum macOS + Linux)
- [ ] Verify Phase 3-5 compatibility (follow-up review)
