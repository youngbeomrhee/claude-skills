# claude-md-creator

![Claude Code](https://img.shields.io/badge/Claude_Code-Plugin-D97757?style=flat-square&logo=claude&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)
![Version](https://img.shields.io/badge/Version-1.0.0-green?style=flat-square)
![No Dependencies](https://img.shields.io/badge/Dependencies-None-brightgreen?style=flat-square)

> Create, evaluate, and improve CLAUDE.md files using Claude-native tools only. No Python, no Bash, no Node.js — just Read, Glob, and Grep.

## Why This Plugin Exists

CLAUDE.md files are written in natural language for Claude to read. But the original evaluation system used **1,627 lines of Python regex** to "understand" that natural language — a paradigm mismatch that required 7 auto-refine iterations and extensive edge-case handling:

| Problem | Cause |
|---------|-------|
| Code block fence tracking (`` ``` `` vs `~~~`) | Regex can't understand context |
| `"like React"` comparison context filtering | Regex can't understand meaning |
| 30+ built-in command exclusion list | Regex can't judge "user-defined script" |
| PEP 735, Poetry, PEP 621 parsing | Each dependency format needs a dedicated parser |
| Monorepo sub-package depth limits | Deterministic filesystem traversal required |

**The insight**: Claude already understands all of this natively. When Claude reads a CLAUDE.md with the Read tool, it naturally distinguishes code block examples from actual instructions, comparison context from real framework references, and built-in commands from user scripts — no regex needed.

## Features

| Mode | Description | Time |
|------|-------------|------|
| **Create** | Generate CLAUDE.md by analyzing codebase + interviewing user | ~3-5 min |
| **Quick Eval** | Verify every claim against the actual codebase | ~1-2 min |
| **Eval All** | Recursively find and evaluate all CLAUDE.md + `.claude/rules/` files | ~2-5 min |
| **Full Eval** | Quick Eval + blind A/B effectiveness test | ~5-10 min |
| **GitHub Eval** | Evaluate any public/private repo's CLAUDE.md | ~2-3 min |
| **Improve** | Auto-fix stale, vague, or conflicting instructions | ~2-3 min |
| **Auto-Refine** | Self-improving evaluation prompt pipeline | N cycles |

### Evaluation Pipeline

```
CLAUDE.md (single)              All CLAUDE.md files (recursive)
    │                               │
    ▼                               ▼
Step 1: Quantitative Check      Discovery: **/CLAUDE.md, .claude/rules/*.md
    │                               │
    ▼                               ▼
Step 2: Semantic Check          Per-file Quick Eval (parallel)
    │   8 claim types × 6 status   │
    │   Conflict + Anti-pattern     ▼
    ▼                           Cross-file Analysis
evaluation_result.json              │  Conflicts, redundancy,
                                    │  hierarchy gaps, scope overlap
                                    ▼
                                Unified Report (per-file + project avg)
```

### 8 Claim Types

| Type | Example | Verification |
|------|---------|-------------|
| `command` | `` `pnpm dev` — start dev server `` | package.json scripts, Makefile targets |
| `path_reference` | `API handlers in src/api/handlers/` | Glob/Read existence check |
| `framework_reference` | `Uses Prisma for database access` | Dependency files |
| `convention` | `Named exports only` | Code sampling (2-3 files) |
| `naming_pattern` | `Components use PascalCase` | File/variable name sampling |
| `env_reference` | `NEXT_PUBLIC_* vars in client bundle` | .env.example files |
| `vague` | `Write clean, maintainable code` | Flag + suggest concretization |
| `generic` | `Follow DRY principle` | Flag + suggest removal |

## Installation

```bash
# 1. Add marketplace (one-time)
/plugin marketplace add youngbeomrhee/skills

# 2. Install plugin
/plugin install claude-md-creator@yb-skills
```

<details>
<summary><b>Local development install</b></summary>

```bash
git clone https://github.com/youngbeomrhee/skills.git
claude plugin install --plugin-dir ./skills/plugins/claude-md-creator
```

</details>

## Quick Start

```bash
# Create a CLAUDE.md for your project
/claude-md-creator create

# Evaluate your existing CLAUDE.md
/claude-md-creator eval

# Evaluate ALL CLAUDE.md files in the project (recursive)
/claude-md-creator eval all

# Evaluate a GitHub repo's CLAUDE.md
/claude-md-creator eval https://github.com/owner/repo

# Auto-improve your CLAUDE.md
/claude-md-creator improve
```

## Available Skills

| Skill | Command | Description |
|-------|---------|-------------|
| **claude-md-creator** | `/claude-md-creator` | Main skill — create, evaluate, improve CLAUDE.md |
| **auto-refine** | `/claude-md-creator:auto-refine` | Self-improving evaluation prompt pipeline |
| **auto-refine-status** | `/claude-md-creator:auto-refine-status` | View auto-refine progress and history |

## Benchmarks

### Golden Set: 75% → 100% in 4 Iterations

The auto-refine pipeline ran 4 cycles against 8 golden-set test scenarios, reaching 100% pass rate:

```
iter 1: ██████░░░░ 75.0%  (6/8)
iter 2: ████████░░ 87.5%  (7/8)
iter 3: ██████░░░░ 75.0%  (6/8) ← regression
iter 4: ██████████ 100%   (8/8) ✅
```

| # | Scenario | Description | Final |
|---|----------|-------------|-------|
| 1 | minimal-valid | Basic CLAUDE.md with few claims | ✅ |
| 2 | stale-paths | References to non-existent paths | ✅ |
| 3 | vague-generic | Mix of vague and generic instructions | ✅ |
| 4 | code-block-context | Claims inside code blocks (must not extract) | ✅ |
| 5 | conflict-detection | Contradictory instructions | ✅ |
| 6 | over-200-lines | File exceeding recommended length | ✅ |
| 7 | comparison-context | "like React" should not be a framework claim | ✅ |
| 8 | builtin-commands | `npm install` should not be a command claim | ✅ |

### Real-World: 10 GitHub Repos, Average B- (75.3)

After achieving 100% on the golden set, the improved prompts were tested against 10 real open-source repositories:

| # | Repository | Ecosystem | Lines | Score | Grade |
|---|-----------|-----------|-------|-------|-------|
| 1 | caoccao/swc4j | Rust/Java (JNI) | 183 | 94.0 | **A** |
| 2 | nanasess/setup-chromedriver | TypeScript/Actions | 50 | 88.0 | **B+** |
| 3 | spring-projects/spring-ai-examples | Java/Spring | 286 | 82.5 | **B** |
| 4 | modelcontextprotocol/python-sdk | Python | 161 | 82.4 | **B** |
| 5 | flawstick/lts-csp | Next.js/TypeScript | 173 | 76.5 | **B** |
| 6 | p-wegner/coding-aider | Kotlin/IntelliJ | 193 | 76.5 | **B** |
| 7 | snowplow/snowplow-dotnet-tracker | C#/.NET | 389 | 72.0 | **C** |
| 8 | hughjonesd/huxtable | R | 115 | 70.1 | **C** |
| 9 | rpmuller/pyquante2 | Python | 98 | 67.1 | **C** |
| 10 | open-responses/open-responses | Go/Python/JS | 308 | 44.3 | **D** |

**Grade distribution**: A (10%), B+ (10%), B (40%), C (30%), D (10%)

#### Cross-Repo Patterns

| Pattern | Frequency | Description |
|---------|-----------|-------------|
| Stale path references | High | Paths are the most drift-prone claim type |
| Version constraint drift | Medium | Dependency versions change faster than docs |
| Excessive generic claims | Medium | "Follow best practices" dilutes the score |
| Internal contradictions | Low | Conflicting settings within the same file |

## Development Story

### The Paradigm Shift

The original evaluation system had two Python scripts totaling **1,627 lines**:

- `extract_claims.py` (861 lines) — regex-based claim extraction with state machines for code block tracking, comparison context filtering, and 30+ built-in command exclusions
- `verify_coherence.py` (766 lines) — parsers for 6 dependency file formats, monorepo sub-package scanning, `os.path.exists()` calls, and tsconfig alias resolution

All of this was replaced by **Claude-native evaluation** — agent prompts that instruct Claude to Read the CLAUDE.md, identify claims through natural language understanding, and verify them against the codebase using Glob/Read/Grep.

### What Auto-Refine Discovered

The auto-refine pipeline revealed 4 key failure modes and their fixes:

| Failure Mode | Root Cause | Fix |
|-------------|-----------|-----|
| **vague/generic misclassification** | Only 2-type definitions, no boundary criteria | 4-step decision tree: Specific Value → Named Principle → Universal Behavior → Quality Adjective |
| **Claim separation gaps** | "Svelte (SvelteKit)" merged as one claim | Added rule: parenthesized names are independent claims |
| **Non-standard status values** | Status values listed loosely in parentheses | Strict 6-value enum table + explicit prohibition list |
| **Type-status inconsistency** | convention claim assigned "generic" status | Type-status consistency matrix: convention → verified/stale/partial/unverifiable only |

### Key Lessons

1. **Simple priority rules backfire** — "when in doubt, prefer generic" caused over-correction and regression (iter 2→3)
2. **Enums need explicit constraints** — listing values in parentheses is insufficient; tables + prohibition lists are required
3. **Type constrains status** — without explicit type→status mapping, logical contradictions emerge
4. **Regression testing is essential** — fixing one scenario can break another (iter 3 regression)

## Plugin Structure

```
claude-md-creator/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   ├── claude-md-creator/       # Main skill (create, eval, improve)
│   ├── auto-refine/             # Self-improving prompt pipeline
│   └── auto-refine-status/      # Progress viewer
├── agents/
│   ├── native-evaluator.md      # Core evaluation agent
│   ├── github-evaluator.md      # GitHub repo evaluation
│   ├── blind-coder.md           # A/B test coding agent
│   └── blind-judge.md           # Blind comparison judge
├── references/
│   ├── eval-rubric.md           # Evaluation criteria
│   ├── claim-taxonomy.md        # Claim type definitions
│   ├── output-schema.json       # JSON output schema
│   └── prompt-refine.md         # Refinement strategy
├── evals/
│   └── evals.json               # 8 golden-set test scenarios
├── results/                     # Auto-refine iteration results
└── docs/
    └── SPEC-native-eval.md      # Architecture specification
```

## Scoring

```
Quick Eval:  overall = quantitative × 0.3 + coherence × 0.7
Full Eval:   overall = coherence × 0.5 + effectiveness × 0.5
```

| Score | Grade | Meaning |
|-------|-------|---------|
| 90-100 | A | Excellent — all claims verified |
| 80-89 | B | Good — minor issues |
| 70-79 | C | Adequate — several fixes needed |
| 60-69 | D | Needs work — significant gaps |
| 0-59 | F | Major rewrite recommended |

## Requirements

- Claude Code 1.0.33+ (plugin support)
- No external runtime dependencies (Python, Node.js, Bash not required)
- Cross-platform: macOS, Linux, Windows

## License

MIT
