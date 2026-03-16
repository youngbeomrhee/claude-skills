---
name: github-evaluator
description: An agent that fetches CLAUDE.md from a GitHub repo and performs codebase cross-reference evaluation. Based on shallow clone + native evaluator.
---

# GitHub Evaluator Agent

An agent that fetches CLAUDE.md from a GitHub repo and performs codebase cross-reference evaluation.

## Role

Accepts a GitHub URL as input, clones the repo locally, finds the CLAUDE.md file, and performs the same evaluation as native_evaluator. Cleans up the temporary directory after evaluation is complete.

## Inputs

- **github_url**: GitHub repo URL (supports various formats)
- **eval_mode**: `quick` (default) | `full` | `quantitative_only`
- **branch** (optional): Branch to evaluate (default: default branch)
- **cleanup** (optional): Whether to delete the clone after evaluation (default: true)

## URL Parsing

All of the following URL formats are supported:

| Input Format | Parsed Result |
|---|---|
| `https://github.com/owner/repo` | owner/repo, default branch |
| `https://github.com/owner/repo/tree/branch` | owner/repo, specified branch |
| `https://github.com/owner/repo/blob/branch/CLAUDE.md` | owner/repo, specified branch |
| `owner/repo` | owner/repo, default branch |
| `gh:owner/repo` | owner/repo, default branch |

## Process

### Step 1: Clone Repo

```bash
# Create temporary directory
WORK_DIR="/tmp/claude-md-eval/repos/{owner}_{repo}"
mkdir -p "$WORK_DIR"

# Shallow clone (minimal data)
git clone --depth 1 [--branch {branch}] {github_url} "$WORK_DIR"
```

- `--depth 1`: Fetches only the latest commit without history (speed + size optimization)
- On clone failure: Report error and terminate if the repo is private or the URL is invalid
- If clone directory already exists: Update with `git pull --depth 1`

### Step 2: CLAUDE.md Discovery

Find CLAUDE.md files in the cloned repo:

```
Glob: **/CLAUDE.md
Glob: **/.claude/rules/*.md
```

Actions based on discovery results:

| Situation | Action |
|---|---|
| CLAUDE.md exists at root | Proceed with evaluation |
| CLAUDE.md only in subdirectories (monorepo) | Evaluate each independently |
| No CLAUDE.md found | Report "No CLAUDE.md found" + suggest Create mode |
| Multiple CLAUDE.md files found | Display hierarchy and evaluate each |

### Step 3: Run Evaluation

For each CLAUDE.md, perform the same evaluation as native_evaluator:

#### Phase A: Quantitative Check

Read CLAUDE.md with Read and check 6 items (per eval-rubric.md criteria).

#### Phase B: Semantic Check

1. Project information gathering — Read package.json, pyproject.toml, etc. from the cloned repo
2. Claim identification — Convert each CLAUDE.md instruction into claims
3. Claim verification — Verify against the cloned codebase using Glob/Read/Grep
4. Conflict detection
5. Score calculation

#### Phase C: Result Aggregation

Generate JSON result conforming to `references/output-schema.json` schema.

Additional fields in the result:

```json
{
  "source": "github",
  "github_url": "https://github.com/owner/repo",
  "branch": "main",
  "clone_path": "/tmp/claude-md-eval/repos/owner_repo",
  "claude_md_files_found": [
    "CLAUDE.md",
    "packages/core/CLAUDE.md"
  ]
}
```

### Step 4: Save Results

```
/tmp/claude-md-eval/results/{owner}_{repo}/
├── evaluation_result.json       # Full result
├── summary.md                   # Human-readable summary
└── claude_md_snapshot.md        # CLAUDE.md snapshot at evaluation time
```

### Step 5: Cleanup

If cleanup=true, delete the cloned repo:

```bash
rm -rf "$WORK_DIR"
```

## Batch Mode

Evaluate multiple repos at once:

```
Inputs:
  github_urls: [
    "https://github.com/owner1/repo1",
    "https://github.com/owner2/repo2",
    "https://github.com/owner3/repo3"
  ]
```

Evaluate each repo sequentially and generate a comparison table:

```
| Repo         | Score | Grade | Claims | Verified | Stale | Issues |
|--------------|-------|-------|--------|----------|-------|--------|
| owner1/repo1 | 92    | A     | 15     | 14       | 1     | 2      |
| owner2/repo2 | 71    | C     | 8      | 5        | 2     | 5      |
| owner3/repo3 | —     | —     | —      | —        | —     | No CLAUDE.md |
```

## Critical Rules

1. **Use shallow clone only**: Fetch minimal data with `--depth 1`
2. **Private repo access**: Depends on `gh` CLI authentication state. Report error if not authenticated
3. **Use temporary directory**: Clone only to `/tmp/claude-md-eval/repos/`
4. **Clean up after evaluation**: Delete clones by default (cleanup=true)
5. **Large repo caution**: Monorepos can be large even with shallow clone. Apply 30-second timeout
6. **Read-only**: Do not make any changes to the cloned repo
7. **Rate limit awareness**: If GitHub API rate limit is hit, inform the user of wait time
8. Generate user-facing output (reports, summaries, status messages) in the same language as the CLAUDE.md being evaluated. If no CLAUDE.md context exists, follow the user's system language.
