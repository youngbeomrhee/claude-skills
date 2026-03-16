# Auto-Refine Improvement Report

**Target**: `claude-md-creator` native evaluation prompt
**Period**: 2026-03-15, 4 iterations
**Golden Set**: 8 scenarios (`evals.json`)

---

## Overall Results

```
iter 1: ██████░░░░ 75.0%  (6/8)
iter 2: ████████░░ 87.5%  (7/8)
iter 3: ██████░░░░ 75.0%  (6/8) ← regression
iter 4: ██████████ 100%   (8/8) ✅ complete
```

---

## Per-Scenario Improvement Tracking

| # | Scenario | iter 1 | iter 2 | iter 3 | iter 4 | Result |
|---|----------|--------|--------|--------|--------|--------|
| 1 | minimal-valid | ✅ | ✅ | ❌ | ✅ | regression→recovered |
| 2 | stale-paths | ✅ | ❌ | ✅ | ✅ | regression→recovered |
| 3 | vague-generic | ❌ | ✅ | ❌ | ✅ | improved |
| 4 | code-block-context | ✅ | ✅ | ✅ | ✅ | stable |
| 5 | conflict-detection | ✅ | ✅ | ✅ | ✅ | stable |
| 6 | over-200-lines | ✅ | ✅ | ✅ | ✅ | stable |
| 7 | comparison-context | ❌ | ✅ | ✅ | ✅ | improved |
| 8 | builtin-commands | ✅ | ✅ | ✅ | ✅ | stable |

---

## Discovered Failure Modes & Fixes

### 1. vague/generic Misclassification (iter 1, 3)

**Scenario**: `vague-generic`

| Instruction | Expected | iter 1 | iter 3 | iter 4 | Result |
|-------------|----------|--------|--------|--------|--------|
| "Write clean, maintainable code" | vague | vague ✅ | generic ❌ | vague ✅ | ✅ |
| "Follow best practices" | generic | vague ❌ | generic ✅ | generic ✅ | ✅ |
| "Keep functions small and focused" | vague/generic | vague ✅ | generic ✅ | generic ✅ | ✅ |
| "Follow DRY principle" | generic | generic ✅ | generic ✅ | generic ✅ | — |
| "Use meaningful variable names" | generic | vague ❌ | generic ✅ | generic ✅ | ✅ |

**Before** (original prompt):

```
| vague   | Instruction too ambiguous to verify |
| generic | General advice Claude already knows |
```

Only type definitions existed with no criteria for distinguishing edge cases.

**After** (4-step decision tree):

```
1. Specific Value Test → convention/naming_pattern
2. Named Principle Test → generic ("DRY", "best practices", "SOLID")
3. General Behavior Test → generic ("meaningful names", "small functions")
4. Quality Adjective Test → vague ("clean", "maintainable", "proper")
```

**Rationale**: Switching from simple 2-type definitions to a 4-step decision tree ensured consistency on edge cases. Simple rules like "when in doubt, prefer generic" caused over-correction (root cause of iter 2→3 regression), so they were removed.

---

### 2. Missing Claim Separation (iter 1)

**Scenario**: `comparison-context`

| Original | Before | After | Result |
|----------|--------|-------|--------|
| "Framework: Svelte (SvelteKit)" | Merged into single fw-1 | fw-1: Svelte + fw-2: SvelteKit separated | ✅ |

**Before**:

```
- Split when a sentence contains multiple facts
```

**After**:

```
- Split when a sentence contains multiple facts
- Names inside parentheses should also be separated as independent claims
  (e.g., "Svelte (SvelteKit)" → fw-1: Svelte + fw-2: SvelteKit)
```

---

### 3. Non-Standard Status Values (iter 2)

**Scenario**: `stale-paths`

| Claim | Expected | iter 2 Actual | iter 4 | Result |
|-------|----------|---------------|--------|--------|
| `src/api/routes/` | stale | "unverified" ❌ | stale ✅ | ✅ |
| `lib/utils/` | stale | "unverified" ❌ | stale ✅ | ✅ |

**Before**:

```
Record status (verified/stale/partial/unverifiable/vague/generic) and evidence for each claim.
```

Allowed values were only listed in parentheses with weak constraints, leading the evaluator to use arbitrary values like "unverified" or "not_applicable".

**After**:

```
| Status        | Meaning                          | Condition                              |
|---------------|----------------------------------|----------------------------------------|
| verified      | Confirmed in codebase            | Command/path/framework actually exists |
| stale         | Referenced item does not exist   | Path missing, command missing, dep missing |
| partial       | Only partially matching          | Some elements match, others don't      |
| unverifiable  | Cannot confirm from code alone   | Convention etc. structurally unverifiable |
| vague         | Ambiguous instruction            | Only for vague-type claims             |
| generic       | General advice                   | Only for generic-type claims           |

Values not in the above 6 (e.g., "unverified", "not_applicable", "unknown") are strictly prohibited.
```

---

### 4. Type-Status Mismatch (iter 3)

**Scenario**: `minimal-valid`

| Claim | Type | Expected Status | iter 3 Actual | iter 4 | Result |
|-------|------|-----------------|---------------|--------|--------|
| "Use 2-space indentation" | convention | verified/unverifiable | generic ❌ | unverifiable ✅ | ✅ |

**Before**: No consistency rules between type and status, allowing logical contradictions like assigning "generic" status to a convention-type claim.

**After**:

```
| Claim Type                            | Allowed Status                         |
|---------------------------------------|----------------------------------------|
| command, path, framework, convention… | verified, stale, partial, unverifiable |
| vague                                 | vague (fixed)                          |
| generic                               | generic (fixed)                        |

Violation prohibited: assigning "generic" status to a convention-type claim is a type-status mismatch.
```

---

## Modified Files Summary

| File | Change | Iteration |
|------|--------|-----------|
| `references/claim-taxonomy.md` | Added 4-step vague/generic decision criteria | iter 1, 3 |
| `references/claim-taxonomy.md` | Added type-status consistency rule table | iter 3 |
| `agents/native-evaluator.md` | Added parenthetical name separation rule | iter 1 |
| `agents/native-evaluator.md` | Added allowed status table (6 values) + prohibition list | iter 2 |

---

## Lessons Learned

1. **Simple priority rules are dangerous**: "When in doubt, prefer generic" caused over-correction → multi-step decision trees are more stable
2. **Enumerations need explicit constraints**: Listing status values in parentheses is insufficient → table + prohibition list required
3. **Type and status are not independent**: Without the constraint that type determines the allowed range of status values, logical contradictions arise
4. **Regression testing is essential**: Regression from 87.5% to 75% occurred in iter 2→3 — fixing one failure can break other scenarios

---

## Real-World GitHub Evaluation Results

After achieving 100% on the golden set, the improved prompt was used to evaluate CLAUDE.md files from 10 real GitHub open-source repositories.

### Evaluation Targets & Results

| # | Repo | Ecosystem | Lines | Overall | Grade |
|---|------|-----------|-------|---------|-------|
| 1 | caoccao/swc4j | Rust/Java (JNI) | 183 | 94.0 | **A** |
| 2 | nanasess/setup-chromedriver | TypeScript/Actions | 50 | 88.0 | **B+** |
| 3 | spring-projects/spring-ai-examples | Java/Spring | 286 | 82.5 | **B** |
| 4 | modelcontextprotocol/python-sdk | Python | 161 | 82.4 | **B** |
| 5 | flawstick/lts-csp | Next.js/TypeScript | 173 | 76.5 | **B** |
| 6 | p-wegner/coding-aider | Kotlin/IntelliJ | 193 | 76.5 | **B** |
| 7 | snowplow/snowplow-dotnet-tracker | C#/.NET | 389 | 72.0 | **C** |
| 8 | hughjonesd/huxtable | R | 115 | 70.1 | **C** |
| 9 | rpmuller/pyquante2 | Python | 98 | 67.1 | **C** |
| 10 | open-responses/open-responses | Go/Python/JS hybrid | 308 | 44.3 | **D** |

**Average Score**: 75.3 (B-)

### Key Findings

#### High Scores (A~B+)

- **swc4j (A, 94.0)**: All verifiable claims passed. Build commands, paths, and architecture descriptions matched the actual codebase precisely. Nearly zero vague/generic claims.
- **setup-chromedriver (B+, 88.0)**: Concise at 50 lines with 100% coherence. All claims verified. However, low volume meant some essential sections were missing.

#### Mid-Range Scores (B~C)

- **python-sdk (B, 82.4)**: Generally accurate, but an internal contradiction was detected between `line-length = 120` and `max-line-length = 88`. 4 generic claims present.
- **coding-aider (B, 76.5)**: Detailed IntelliJ plugin architecture description, but some path references were uncertain.
- **snowplow-dotnet-tracker (C, 72.0)**: Excessively long at 389 lines with a high ratio of unverifiable claims.

#### Low Scores (D)

- **open-responses (D, 44.3)**: 308 lines, Go/Python/JS hybrid stack. Only 2 of 9 claims were verifiable. Most path/framework references did not match the actual codebase.

### Cross-Repo Pattern Analysis

| Pattern | Frequency | Description |
|---------|-----------|-------------|
| path_reference verification failure | High | Path references are the claim type most prone to becoming stale |
| Version constraint staleness | Medium | Dependency versions change rapidly, diverging from documented versions |
| Excessive generic claims | Medium | Instructions like "Follow best practices" dilute the score |
| Internal contradictions | Low | Conflicting settings within the same document (e.g., python-sdk's line-length) |

### Grade Distribution

```
A  : █          (1, 10%)
B+ : █          (1, 10%)
B  : ████       (4, 40%)
C  : ███        (3, 30%)
D  : █          (1, 10%)
```

### Conclusion

The improved prompt operates consistently across diverse ecosystems (Python, Java, .NET, R, TypeScript, Rust/Java, Go/Python/JS). It can perform meaningful evaluations not only on internal golden set scenarios but also on real-world open-source CLAUDE.md files.
