# Claim Taxonomy

Definition of 8 types for classifying each instruction (claim) in CLAUDE.md.

## Claim Types

### Verifiable Types

| Type | ID Prefix | Description | Example | Verification |
|------|-----------|-------------|---------|-------------|
| **command** | `cmd-` | Executable CLI command within backticks | `` `pnpm dev` — start dev server `` | Verify existence in package.json scripts, Makefile targets, Cargo.toml, etc. |
| **path_reference** | `path-` | File or directory path reference | `API handlers live in src/api/handlers/` | Verify actual existence using Glob/Read |
| **framework_reference** | `fw-` | Library/framework name | `Uses Prisma for database access` | Verify in dependency files (package.json, pyproject.toml, etc.) |
| **convention** | `conv-` | Coding rule, style guide | `Named exports only (no default exports)` | Verify actual adherence through code sampling |
| **naming_pattern** | `name-` | Naming convention | `Components use PascalCase` | Verify through file/variable name sampling |
| **env_reference** | `env-` | Environment variable reference | `NEXT_PUBLIC_* vars are in client bundle` | Verify in .env.example, .env.sample, etc. |

### Non-Verifiable Types (Flagging)

| Type | ID Prefix | Description | Example | Action |
|------|-----------|-------------|---------|--------|
| **vague** | `vague-` | Instruction with project-specific intent but overly ambiguous wording | `Write clean, maintainable code` | Recommend concretization |
| **generic** | `generic-` | General programming principle with no project-specific value | `Follow DRY principle` | Recommend removal (token waste) |

#### Criteria for Distinguishing vague vs generic

Distinguishing these two types is critical. Apply the following tests in order:

1. **Specific Value Test**: Does this instruction contain specific values/names/settings?
   - YES → **convention** or **naming_pattern** (e.g., "Use 2-space indentation", "Named exports only", "PascalCase for components")
   - NO → Proceed to next test

2. **Named Principle Test**: Does this instruction reference a universally named programming principle?
   - YES → **generic** (e.g., "Follow DRY principle", "Follow best practices", "Follow SOLID principles")
   - NO → Proceed to next test

3. **General Behavior Test**: Is this instruction a general behavioral guideline found in programming textbooks?
   - YES → **generic** (e.g., "Use meaningful variable names", "Keep functions small and focused", "Write unit tests")
   - NO → Proceed to next test

4. **Ambiguous Quality Adjective Test**: Does this instruction use undefined quality adjectives like "clean", "maintainable", "proper", "good", "organized"?
   - YES → **vague** (e.g., "Write clean, maintainable code", "Handle errors properly", "Keep the codebase organized")
   - NO → Classify as another type

**Key Differences**:
- **convention**: Project rules with specific values ("2-space", "PascalCase", "named exports")
- **generic**: Named or universally known programming principles ("DRY", "best practices", "meaningful names")
- **vague**: Ambiguous instructions composed of undefined quality adjectives ("clean code", "proper handling")

#### Type-Status Consistency Rules

The allowed status values are determined by the claim's type:

| Claim Type | Allowed Status |
|------------|------------|
| command, path_reference, framework_reference, convention, naming_pattern, env_reference | verified, stale, partial, unverifiable |
| vague | vague (fixed) |
| generic | generic (fixed) |

**Violation prohibited**: Assigning "generic" status to a convention type claim, or assigning "verified" status to a generic type claim, constitutes a type-status mismatch.

## Status Values

Verification result for each claim:

| Status | Meaning | Score Impact |
|--------|---------|-------------|
| **verified** | Confirmed in the codebase | Positive contribution to score |
| **stale** | The referenced item does not exist | Negative contribution to score |
| **partial** | Only partially matching | Negative contribution to score |
| **unverifiable** | Cannot be confirmed from code alone | Excluded from score calculation |
| **vague** | Ambiguous instruction (non-verifiable type) | Excluded from score calculation, reported as issue |
| **generic** | General advice (non-verifiable type) | Excluded from score calculation, reported as issue |

## Scoring

```
coherence_score = verified / (verified + stale + partial) × 100
```

- unverifiable, vague, generic are excluded from the denominator
- If there are fewer than 3 verifiable claims, unverifiable claims are included with 0.5 weight for adjustment

## Grade Scale

| Score | Grade | Meaning |
|-------|-------|---------|
| 90-100 | A | Excellent — all claims match the codebase |
| 80-89 | B | Good — a few minor issues |
| 70-79 | C | Adequate — several corrections needed |
| 60-69 | D | Needs work — significant gaps |
| 0-59 | F | Major rewrite recommended |
