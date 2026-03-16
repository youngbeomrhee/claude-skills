# Evaluation Rubric

Evaluation criteria for CLAUDE.md based on the official documentation (https://code.claude.com/docs/en/memory).

---

## Step 1: Quantitative Check

Read the CLAUDE.md file using Read, then verify the following items.

### 1.1 Size

- **Criteria**: 200 lines or fewer
- **Rationale**: "Target under 200 lines per CLAUDE.md file. Longer files consume more tokens and reduce adherence."
- **Scoring**: 200 lines or fewer = pass, exceeding = warn (reported as issue)

### 1.2 Structure

- **Criteria**: Are markdown headers (`#`, `##`, `###`) and bullets (`-`, `*`) used?
- **Rationale**: "Use markdown headers and bullets to group related instructions."
- **Scoring**: At least 1 header + at least 1 bullet = pass

### 1.3 Commands

- **Criteria**: Are executable commands included within backticks?
- **Rationale**: Best practice examples provide build/test/lint commands in backticks
- **Scoring**: At least 1 backtick command = pass

### 1.4 Import Validity (Imports)

- **Criteria**: If `@path/to/file` syntax exists, does the target file actually exist?
- **Rationale**: "Imported files are expanded and loaded into context at launch."
- **Scoring**: If no @import exists, skip; otherwise verify file existence for each path
- **Note**: Relative paths are resolved relative to the CLAUDE.md file location. Maximum 5 depth levels.

### 1.5 .claude/rules/ Frontmatter Validity

- **Criteria**: Is the `paths:` frontmatter in `.claude/rules/*.md` files a valid glob pattern?
- **Rationale**: "Rules can be scoped to specific files using YAML frontmatter with the paths field."
- **Scoring**: If no rules files exist, skip; otherwise parse frontmatter and validate glob patterns

### 1.6 Essential Sections

- **Criteria**: At least one of Build, Test, or Code Style sections exists
- **Rationale**: Best practice structure presents these 3 as essential sections
- **Scoring**: Detect build/run/test/style/convention related keywords in headers or content

### Quantitative Score Calculation

```
Each item: pass=1, warn/fail=0
score = (number of passed items / total items) × 100
```

Items imports_valid and rules_frontmatter_valid are excluded from scoring if the corresponding elements do not exist.

---

## Step 2: Semantic Check

Identify each instruction in CLAUDE.md as a claim at the semantic level, then cross-validate against the codebase.

### 2.1 Claim Identification

Read CLAUDE.md and convert each instruction into a claim using the following criteria:

- **One verifiable fact = one claim**
- Examples within code blocks are part of the explanation, not claims
- Comparative context like "like React", "similar to Django" is not a framework claim
- Built-in commands (npm install, git add, etc.) are not command claims — only user-defined scripts

### 2.2 Claim Verification

Verification methods by claim type:

#### command

1. Read package.json / Makefile / Cargo.toml / pyproject.toml, etc.
2. Check if the command exists in scripts/targets
3. If it exists, verified; if not, stale

#### path_reference

1. Use Glob to check if the path exists
2. If @/ alias exists, resolve the path from tsconfig.json/jsconfig.json
3. If it exists, verified; if not, stale (provide fix_suggestion if an alternative path exists)

#### framework_reference

1. Read dependency files (package.json, pyproject.toml, Cargo.toml, go.mod, etc.)
2. Check if the framework exists in dependencies
3. Also verify existence of configuration files (next.config.*, tailwind.config.*, etc.)
4. If it exists, verified; if not, stale

#### convention

1. First determine if the rule is specific and verifiable
2. If verifiable: sample 2-3 related code files using Read to check actual adherence
3. If followed, verified; if not followed, partial; if unconfirmable from code alone, unverifiable

#### naming_pattern

1. Search related files/directories using Glob
2. Check if the naming pattern is actually applied
3. If matching, verified; if partially matching, partial; if unconfirmable, unverifiable

#### env_reference

1. Search .env.example, .env.sample, .env.local.example using Glob
2. Check if the environment variable (or pattern) exists
3. If it exists, verified; if not, stale

#### vague

- Instructions that cannot be verified, such as "Write clean code", "Follow best practices"
- status = vague, provide concretization suggestions in fix_suggestion

#### generic

- Advice that Claude already knows, such as "Follow DRY principle", "Use meaningful names"
- status = generic, recommend removal or project-specific customization in fix_suggestion

### 2.3 Conflict Detection

Detect contradictions between instructions:
- "Use default exports" vs "Named exports only"
- "Use semicolons" vs "No semicolons"
- Cases where different rules apply to the same target

### 2.4 Anti-Pattern Detection

Detect anti-patterns from the official documentation:
- Information that can be inferred from code (listing data model fields, etc.)
- Git history or ephemeral information
- Long reference documents included inline
- General programming advice

---

## Efficient Verification Strategy

To prevent Claude from making excessive tool calls:

1. **Batch read dependency files**: Read package.json, pyproject.toml, etc. all at once first, then batch-verify all command/framework claims
2. **Batch process path checks**: Group all path claims and handle them with a single Glob call where possible
3. **Minimize code sampling**: Sample only 2-3 files when verifying convention/naming claims
4. **Never attempt to read non-existent files**: Confirm existence with Glob before using Read
