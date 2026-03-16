---
name: claude-md-creator
description: |
  Create, evaluate, and improve CLAUDE.md files for Claude Code projects. Use this skill whenever
  the user mentions CLAUDE.md, project instructions, Claude Code memory, project rules, .claude/rules/,
  or wants to set up persistent instructions for Claude Code. Also trigger when users say "init my project
  for Claude", "set up Claude Code", "evaluate my CLAUDE.md", "improve my CLAUDE.md", "create project rules",
  "audit my CLAUDE.md", "evaluate all CLAUDE.md", "audit all", "check all CLAUDE.md files",
  or ask about best practices for Claude Code memory. Also trigger when users provide
  a GitHub URL and ask to evaluate its CLAUDE.md, or say "evaluate this repo's CLAUDE.md",
  "review this repo's CLAUDE.md", "check this repo's CLAUDE.md". This skill handles creating new CLAUDE.md files,
  evaluating existing ones (local or from GitHub), evaluating all CLAUDE.md files recursively, and improving them.
---

# CLAUDE.md Creator & Evaluator

Create well-structured CLAUDE.md files and evaluate existing ones against the actual codebase using Claude-native tools only. No external scripts (Python, Bash, Node.js) required.

## Overview

CLAUDE.md files give Claude persistent instructions for a project. They're loaded into every session's context window and directly affect how reliably Claude follows your team's standards. This skill provides:

1. **Create** — Generate a CLAUDE.md from scratch by interviewing the user and analyzing the codebase
2. **Quick Eval** — Verify that every claim in your CLAUDE.md matches the actual codebase (~1-2 min)
3. **Full Eval** — Quick Eval + blind A/B test proving your CLAUDE.md actually changes Claude's behavior (~5-10 min)
4. **Eval All** — Recursively find and evaluate all CLAUDE.md and .claude/rules/ files in the project (~2-5 min)
5. **GitHub Eval** — Fetch a GitHub repo's CLAUDE.md and evaluate it against its codebase (~2-3 min)
6. **Improve** — Apply evaluation findings to fix stale, vague, or ineffective instructions

## Mode Detection

- **"Create CLAUDE.md"** / **"Init for Claude"** → Create mode
- **"Evaluate CLAUDE.md"** / **"Audit"** / **"Quick eval"** → Quick Eval (root CLAUDE.md only)
- **"Evaluate all"** / **"Audit all"** / **"Check all CLAUDE.md"** / **"Evaluate all CLAUDE.md files"** → Eval All (recursive)
- **"Full eval"** / **"Blind test"** / **"Test effectiveness"** → Full Eval
- **GitHub URL + "eval"** / **"Evaluate this repo's CLAUDE.md"** / **"Review this repo's CLAUDE.md"** → GitHub Eval
- **"Improve CLAUDE.md"** / **"Fix my CLAUDE.md"** → Quick Eval → Improve
- Ambiguous → Ask the user

---

## Create Mode

### Step 1: Analyze the Codebase

Scan the project to gather context using Read, Glob, and Grep:

1. **Package manager & scripts**: Read `package.json`, `pyproject.toml`, `Cargo.toml`, `Makefile`, etc.
2. **Framework detection**: Check for Next.js, React, Vue, Django, Rails, etc.
3. **Test framework**: Look for jest, pytest, vitest, etc. and how to run tests
4. **Linting/formatting**: Check for eslint, prettier, ruff, black, etc.
5. **Directory structure**: Glob top-level and key directories (`src/`, `app/`, `lib/`, `tests/`)
6. **Existing configuration**: Check for `.claude/`, `.claude/rules/`, existing CLAUDE.md
7. **Git conventions**: Read recent commit messages for style patterns
8. **CI/CD**: Check for `.github/workflows/`, `Jenkinsfile`, etc.

### Step 2: Interview the User

Fill gaps that can't be determined from code alone:

1. **Build & deploy**: Any non-obvious build steps? Environment-specific commands?
2. **Architecture decisions**: Patterns the team follows that aren't obvious from code?
3. **Coding standards**: Naming conventions, import ordering, error handling patterns?
4. **Common workflows**: Feature branch strategy? PR process?
5. **Gotchas**: Known pitfalls or anti-patterns specific to this project?
6. **Team context**: Monorepo? Multiple teams with different conventions?

Don't ask about things already found from the codebase scan.

### Step 3: Generate the CLAUDE.md

#### Size: Target under 200 lines

The entire CLAUDE.md is loaded into every session's context window. Longer files consume more tokens and reduce adherence. Split using `.claude/rules/` files.

#### Structure: Markdown headers and bullets

Claude scans structure the way readers do. Organized sections are easier to follow.

#### Specificity: Concrete and verifiable

- "Use 2-space indentation" instead of "Format code properly"
- "Run `npm test` before committing" instead of "Test your changes"
- "API handlers live in `src/api/handlers/`" instead of "Keep files organized"

#### Recommended Structure

```markdown
# Project Name

Brief one-line description.

## Build & Run

- `command` — what it does

## Test

- `command` — run all tests

## Code Style

- Concrete rule 1

## Architecture

- Directory conventions

## Conventions

- Naming, imports, error handling

## Common Pitfalls

- Gotcha — why and how to avoid
```

#### What to include

- Build, lint, and test commands (exact commands)
- Code style preferences (indentation, naming, imports)
- Architecture patterns and directory conventions
- Common pitfalls specific to this project
- Workflow instructions (branch naming, PR process)

#### What NOT to include

- Things derivable from reading code
- Git history or recent changes
- Long reference documentation (use `.claude/rules/` or `references/`)
- Ephemeral task details
- Generic programming advice Claude already knows

### Step 4: Generate .claude/rules/ Files (if needed)

If instructions exceed 200 lines, or if the project has distinct domains, split:

```
.claude/
├── CLAUDE.md              # Main instructions (under 200 lines)
└── rules/
    ├── code-style.md      # Code style guidelines
    ├── testing.md         # Testing conventions
    └── api-design.md      # API development rules
```

Path-specific rules use YAML frontmatter:

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API Development Rules

- All endpoints must include input validation
```

### Step 5: Present and Iterate

Show the generated files. Ask for feedback. Adjust.

---

## Quick Eval (Step 1 + Step 2)

Fast codebase coherence check. All evaluation is performed by Claude using native tools — no external scripts.

### Step 1: Quantitative Check

Read the CLAUDE.md and check against official best practices (https://code.claude.com/docs/en/memory):

1. **Size**: Total line count. Warn if over 200 lines.
2. **Structure**: Markdown headers (`#`) and bullets (`-`, `*`) present?
3. **Commands**: Backtick-enclosed CLI commands exist?
4. **@imports**: If `@path` references exist, verify each target file exists (resolve relative to CLAUDE.md location, max 5 hops).
5. **Rules frontmatter**: If `.claude/rules/` files exist, verify `paths:` YAML frontmatter contains valid glob patterns.
6. **Essential sections**: At least one of build/test/code-style sections present?

Score: `pass count / total applicable items × 100`

### Step 2: Semantic Check

Read the CLAUDE.md and identify each instruction as a semantic claim, then verify against the codebase.

#### Step 2a: Gather Project Info (once)

Read project dependency/config files to use for verification:

```
Read (if exists): package.json, pyproject.toml, Cargo.toml, go.mod, Makefile, Gemfile
Glob (if exists): .env.example, .env.sample, .env.local.example
Read (if exists): tsconfig.json, jsconfig.json
```

Skip files that don't exist. This pre-loading avoids repeated reads during claim verification.

#### Step 2b: Identify Claims

Read CLAUDE.md and convert each instruction into a typed claim:

| Claim Type | ID Prefix | What to look for |
|------------|-----------|-----------------|
| command | `cmd-` | Backtick CLI commands (exclude builtins like `npm install`, `git add`) |
| path_reference | `path-` | File/directory paths mentioned |
| framework_reference | `fw-` | Library/framework names (exclude comparison context like "like React") |
| convention | `conv-` | Coding rules ("Use X", "Never Y", "Prefer X over Y") |
| naming_pattern | `name-` | Naming conventions (PascalCase, camelCase, etc.) |
| env_reference | `env-` | Environment variables |
| vague | `vague-` | Too vague to verify ("Write clean code") |
| generic | `generic-` | Generic advice Claude already knows ("Follow DRY") |

Rules:
- Content inside code blocks (``` fences) is example/illustration, not a claim
- One verifiable fact = one claim
- Assign sequential IDs: cmd-1, cmd-2, path-1, etc.

#### Step 2c: Verify Each Claim

Using the project info gathered in Step 2a:

- **command**: Check if the command exists in package.json scripts, Makefile targets, Cargo.toml, etc.
- **path_reference**: Glob/Read to verify existence. Resolve @/ aliases via tsconfig.
- **framework_reference**: Check dependencies in package.json, pyproject.toml, etc. Also check for config files.
- **convention**: Sample 2-3 code files to verify adherence. Mark unverifiable if code-only check is impossible.
- **naming_pattern**: Glob relevant files to check naming patterns.
- **env_reference**: Check .env.example/.env.sample files.
- **vague**: Flag with fix_suggestion for concretization.
- **generic**: Flag with fix_suggestion for removal.

Assign status: `verified` | `stale` | `partial` | `unverifiable` | `vague` | `generic`

#### Step 2d: Detect Conflicts

Compare all claims to find contradictions (e.g., "Use default exports" vs "Named exports only").

#### Step 2e: Calculate Score

```
coherence_score = verified / (verified + stale + partial) × 100
```

- If fewer than 3 verifiable claims: blend unverifiable with 0.5 weight
- Grade: A (90-100), B (80-89), C (70-79), D (60-69), F (0-59)

### Generate Quick Eval Report

Combine quantitative and semantic scores:

```
overall_score = (quantitative_score × 0.3) + (coherence_score × 0.7)
```

Write evaluation result as JSON to `/tmp/claude-md-eval/evaluation_result.json` following the schema in `references/output-schema.json`.

Present the summary to the user:
- Overall score (0-100) with grade (A-F)
- Quantitative check results
- Coherence breakdown (verified / stale / partial / unverifiable / vague / generic)
- Conflicts found
- Top issues ranked by severity
- Fix suggestions with specific corrections

### N=3 Consensus Mode (Optional)

For higher confidence, run the evaluation 3 times with independent agent sessions:

```
Agent(native_evaluator): Round 1 → result_1.json
Agent(native_evaluator): Round 2 → result_2.json
Agent(native_evaluator): Round 3 → result_3.json
```

Consensus logic:
- For each claim, adopt the status that appears in 2+ rounds
- If a round's overall score differs by ±5% from the others, exclude it as an outlier
- If all 3 rounds disagree, adopt the most conservative (lowest score)

Trigger consensus mode when the user says "thorough eval", "precise eval", or "consensus eval".

---

## Eval All (Recursive Project Evaluation)

Find and evaluate every CLAUDE.md and `.claude/rules/` file in the project. Useful for monorepos or projects with nested instruction files.

### Step 1: Discovery

Recursively scan the project for all instruction files:

```
Glob: **/CLAUDE.md
Glob: **/.claude.md
Glob: **/.claude.local.md
Glob: **/.claude/rules/*.md
```

Exclude common non-project directories:
- `node_modules/`, `.git/`, `dist/`, `build/`, `.next/`, `__pycache__/`, `venv/`, `.venv/`, `vendor/`

Present the discovered file list to the user:

```
Found 5 instruction files:
  1. ./CLAUDE.md (root)
  2. ./packages/api/CLAUDE.md
  3. ./packages/web/CLAUDE.md
  4. ./.claude/rules/code-style.md
  5. ./.claude/rules/testing.md
```

### Step 2: Evaluate Each File

Run Quick Eval (Step 1 Quantitative + Step 2 Semantic) independently for each file:

- Each CLAUDE.md is evaluated against **its own scope** — a subdirectory CLAUDE.md is verified against files in that subdirectory, not the whole project
- `.claude/rules/*.md` files: additionally validate `paths:` frontmatter glob patterns and check that referenced paths exist
- Run evaluations in parallel using Agent where possible

### Step 3: Cross-File Analysis

After individual evaluations, detect cross-file issues:

1. **Conflicts**: Contradicting instructions across files (e.g., root says "Use tabs", subdirectory says "Use spaces")
2. **Redundancy**: Same instruction duplicated in multiple files (recommend consolidating to root or removing duplicates)
3. **Hierarchy gaps**: Subdirectory CLAUDE.md references conventions not established in any parent CLAUDE.md
4. **Scope overlap**: `.claude/rules/` file with `paths:` glob that overlaps with a subdirectory CLAUDE.md

### Step 4: Summary Report

Present a unified report:

```
## Project-Wide CLAUDE.md Evaluation

### File Scores

| # | File | Lines | Score | Grade | Verified | Stale | Vague | Generic |
|---|------|-------|-------|-------|----------|-------|-------|---------|
| 1 | ./CLAUDE.md | 85 | 92 | A | 12/13 | 1 | 0 | 0 |
| 2 | ./packages/api/CLAUDE.md | 45 | 78 | C | 5/7 | 1 | 1 | 0 |
| 3 | ./packages/web/CLAUDE.md | 62 | 85 | B | 8/9 | 0 | 0 | 1 |
| 4 | .claude/rules/code-style.md | 30 | 90 | A | 4/4 | 0 | 0 | 0 |
| 5 | .claude/rules/testing.md | 25 | 70 | C | 2/3 | 1 | 0 | 0 |

**Project Average**: 83 (B)
**Total Claims**: 36 (31 verified, 3 stale, 1 vague, 1 generic)

### Cross-File Issues
- ⚠ Conflict: ./CLAUDE.md says "Named exports only" but ./packages/api/CLAUDE.md says "Use default exports for pages"
- ℹ Redundancy: "Run `pnpm test`" appears in both root and packages/api CLAUDE.md
```

### Step 5: Improve All (Optional)

If the user requests improvement after Eval All:

1. Fix issues in priority order: cross-file conflicts first, then per-file issues
2. Consolidate redundant instructions to the appropriate level (root vs subdirectory)
3. Show diffs for each file before applying
4. Re-run Eval All to confirm improvement

---

## GitHub Eval

Evaluate CLAUDE.md files from any public (or authenticated private) GitHub repository.

### Input Formats

All of the following URL formats are accepted:

- `https://github.com/owner/repo`
- `https://github.com/owner/repo/tree/branch`
- `https://github.com/owner/repo/blob/branch/CLAUDE.md`
- `owner/repo` (shorthand)

### Process

#### Step 1: Clone Repository

```bash
WORK_DIR="/tmp/claude-md-eval/repos/{owner}_{repo}"
git clone --depth 1 {url} "$WORK_DIR"
```

- Uses `--depth 1` for minimal data transfer
- For private repos, relies on `gh` CLI authentication
- If clone fails, report error and suggest checking URL/auth

#### Step 2: Find CLAUDE.md Files

```
Glob: **/CLAUDE.md
Glob: **/.claude/rules/*.md
```

| Situation | Action |
|---|---|
| Root CLAUDE.md found | Evaluate directly |
| Subdirectory CLAUDE.md only (monorepo) | Evaluate each independently |
| No CLAUDE.md found | Report "No CLAUDE.md found" |
| Multiple CLAUDE.md files | Show hierarchy, evaluate each |

#### Step 3: Run Evaluation

For each CLAUDE.md found, run the same Step 1 (Quantitative) + Step 2 (Semantic) as Quick Eval, using the cloned codebase as the verification target.

Use `agents/github-evaluator.md` for the full process.

#### Step 4: Present Results

Results include all standard Quick Eval outputs plus:
- GitHub source URL and branch
- List of CLAUDE.md files found
- Snapshot of the evaluated CLAUDE.md

Results saved to `/tmp/claude-md-eval/results/{owner}_{repo}/`.

#### Step 5: Cleanup

Delete the cloned repository after evaluation (default). Pass `--keep` to retain for manual inspection.

### Batch Mode

Evaluate multiple repos at once by providing a list of URLs:

```
Evaluate these repos' CLAUDE.md:
- https://github.com/owner1/repo1
- https://github.com/owner2/repo2
- https://github.com/owner3/repo3
```

Produces a comparison table:

```
| Repo         | Score | Grade | Verified | Stale | Vague | Top Issue          |
|--------------|-------|-------|----------|-------|-------|--------------------|
| owner1/repo1 | 92    | A     | 14/15    | 1     | 0     | Stale path ref     |
| owner2/repo2 | 71    | C     | 5/8      | 2     | 3     | Vague instructions |
| owner3/repo3 | —     | —     | —        | —     | —     | No CLAUDE.md       |
```

---

## Full Eval (Quick Eval + Blind A/B Test)

Everything in Quick Eval plus a blind effectiveness test.

### Quick Eval

Run the same Step 1 + Step 2 as above.

### Phase 3: Dynamic Eval Task Generation

From verified claims, generate 3-7 coding tasks that test whether Claude follows conventions **without being told explicitly**:

```
Agent(eval_generator): Create realistic coding tasks from verified claims.
  Inputs: evaluation_result.json, project_root
  Output: eval_tasks.json with natural task_prompt for each task
```

**Important**: Task prompts must NOT mention CLAUDE.md conventions. The point is to test whether Claude follows them from context alone.

### Phase 4: Blind Effectiveness Test

For each eval task, run two coding sessions:

#### Step 4a: Run WITH CLAUDE.md

```
Agent(blind-coder): Complete coding task with CLAUDE.md context.
  Inputs: task_prompt, project_root, output_dir, claude_md_content
```

#### Step 4b: Run WITHOUT CLAUDE.md

```
Agent(blind-coder): Complete coding task without CLAUDE.md context.
  Inputs: task_prompt, project_root, output_dir
```

Run both in parallel per task.

#### Step 4c: Blind Judgment

Randomly assign outputs as "A" and "B":

```
Agent(blind-judge): Compare Output A vs Output B.
  Inputs: task_prompt, output_a_dir, output_b_dir, project_root, expected_behaviors
  Output: JSON with winner, reasoning, scores
```

### Phase 5: Final Report

Combine all results:

```
effectiveness_score = (wins + ties × 0.5) / total_tasks × 100
overall_score = coherence_score × 0.5 + effectiveness_score × 0.5
```

Present full report:
1. **Coherence** — codebase alignment details
2. **Effectiveness** — blind test wins/ties/losses + score delta
3. **Recommendations** — remove, fix, add, strengthen

---

## Improve Mode

After evaluation, apply improvements:

1. Read the evaluation report (run Quick Eval if not already done)
2. Fix issues in priority order:
   - **Remove** vague/generic instructions
   - **Fix** stale paths and outdated references
   - **Add** missing key sections (build, test commands)
   - **Strengthen** partially-followed conventions
   - **Resolve** conflicts between instructions
3. Show the user a diff of changes
4. If the file needs splitting, create `.claude/rules/` files
5. Re-run Quick Eval to confirm improvement

---

## Scoring & Grades

### Quantitative Score (Step 1)

Percentage of best-practice checks that pass:

```
quantitative_score = pass_count / applicable_items × 100
```

### Coherence Score (Step 2)

Percentage of verifiable claims that match the codebase:

```
coherence_score = verified / (verified + stale + partial) × 100
```

### Effectiveness Score (Full Eval)

Weighted blind test results:

```
effectiveness_score = (wins + ties × 0.5) / total_tasks × 100
```

### Overall Score

| Eval Type | Formula |
|-----------|---------|
| Quick Eval | `quantitative × 0.3 + coherence × 0.7` |
| Full Eval | `coherence × 0.5 + effectiveness × 0.5` |

### Grades

| Score | Grade | Meaning |
|-------|-------|---------|
| 90-100 | A | Excellent — all claims match, high effectiveness |
| 80-89 | B | Good — minor issues |
| 70-79 | C | Adequate — several fixes needed |
| 60-69 | D | Needs work — significant gaps |
| 0-59 | F | Major rewrite recommended |

---

## File Reference

### Agents
- `agents/native-evaluator.md` — Core evaluator: reads CLAUDE.md, verifies claims against codebase
- `agents/github-evaluator.md` — GitHub repo fetch + CLAUDE.md evaluation pipeline
- `agents/blind-coder.md` — Executes coding tasks (with or without CLAUDE.md)
- `agents/blind-judge.md` — Blind comparison of two code outputs

### References
- `references/eval-rubric.md` — Evaluation criteria based on official docs
- `references/claim-taxonomy.md` — Claim type definitions and scoring
- `references/output-schema.json` — JSON schema for evaluation output

### Documentation
- `docs/SPEC-native-eval.md` — Technical specification for the native eval architecture
