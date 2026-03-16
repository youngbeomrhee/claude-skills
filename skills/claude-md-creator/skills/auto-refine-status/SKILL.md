---
name: auto-refine-status
description: |
  Checks the progress and improvement history of the auto-refine pipeline. Reads summary.json to
  display the current pass rate, failed scenarios, prompt modification history, and iteration trends.
  Triggers: "/auto-refine-status", "refine status", "improvement status", "auto refine status",
  "auto-refine progress", "test status", "eval test status"
---

# Auto-Refine Status — Progress Check

Displays the current state of the auto-refine pipeline.

## Usage

```
/claude-md-creator:auto-refine-status
```

---

## Execution Process

**Print the following banner first upon execution:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔧 Skill Activated: auto-refine-status
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Step 1: Load State File

```
Read: results/summary.json
```

If the file does not exist: Print "auto-refine has not been run yet. Start with `/claude-md-creator:auto-refine`." and exit.

Generate user-facing output (reports, summaries, status messages) in the same language as the CLAUDE.md being evaluated. If no CLAUDE.md context exists, follow the user's system language.

### Step 2: Output Progress Summary

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Auto-Refine Status
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Last run: 2026-03-15 13:30 (iter 3)
Pass rate: 6/8 (75%)

📈 Progress trend:
  iter 1: ████░░░░░░ 50% (4/8)
  iter 2: ██████░░░░ 75% (6/8)
  iter 3: ██████░░░░ 75% (6/8)

❌ Failed scenarios:
  - stale-paths: Misjudged path-1 as verified (false_positive)
  - code-block-context: Claim extracted from code block (over_extraction)

🔧 Prompt modification history:
  iter 2: eval-rubric.md — Added scripts key check to command verification
  iter 3: eval-rubric.md — Added exact match condition to path verification

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Step 3: Show Related Commits

Display recent auto-refine related commits:

```bash
git log --oneline --grep="auto-refine" -5
```

---

## Important Rules

1. **Read-only**: This skill only reads state and does not modify anything
2. **Progress trend visualization**: Display pass rate per iteration as a progress bar
3. **Failed scenario highlight**: Clearly display currently failing scenarios and their failure modes
