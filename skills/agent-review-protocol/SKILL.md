---
name: agent-review-protocol
description: "Code review quality enforcement: Three-Agent Review pipeline, cross-check protocol, severity handling, and review prompt templates. Load during code review phases or when completing significant implementations. Triggers: 'review protocol', 'three agent review', 'cross check', '代码审查', '交叉检查', 'code review', 'quality review'."
---

# Agent Review Protocol

Code review quality enforcement for AI-assisted development. This skill defines WHO reviews, WHAT they check, and HOW issues are handled — ensuring no significant code ships without independent verification.

**Companion skills:**
- `init-project-gates` — one-time project setup (hook, AGENTS.md, PROGRESS.md)
- `agent-workflow-rules` — runtime discipline (TDD, plan review, verification)

---

## 1. Cross-Check Rule (⛔ Hard Constraint)

All completed development and documentation MUST be independently verified by a different model/agent before delivery.

| Work type | Required cross-check |
| --- | --- |
| Code (feature / bugfix / refactor) | Different model/agent does code review + runs tests |
| Documentation | Different model/agent checks accuracy, completeness, actionability |

### Acceptable Cross-Check Methods

- Oracle consultation (read-only, high-reasoning)
- Different `task` category agent (e.g., `ultrabrain` reviewing `quick` work)
- Different model entirely (e.g., Codex reviewing Claude's work)

### Non-Negotiable

- Cross-check failure → fix → re-verify. Never skip.
- Self-review by the same agent that wrote the code does NOT count.
- Evidence of cross-check must be available (reviewer output, pass/fail).

---

## 2. When to Use Three-Agent Review vs Simplified Review

### Three-Agent Review (Full Pipeline) — REQUIRED when:

- Changes span ≥3 files
- Involves security, authentication, payment, or data migration
- User explicitly requests review
- Plan Review Gate (🔴) marked the task as critical
- Introduces new architecture patterns or external interfaces

### Simplified Review (Self-assessment + Single Oracle Check) — ALLOWED when:

- Single file, ≤20 lines of simple modification
- Pure configuration change (no business logic)
- Documentation / comment changes only

**When in doubt, use Full Pipeline.**

---

## 3. Three-Agent Review Pipeline

Strict sequential execution. No skipping or merging roles.

### Role 1: Implementer (Self-Assessment)

The agent that wrote the code produces a self-assessment report:

```markdown
## Self-Assessment Report

### Changes Summary
- [file:lines] — what was changed and why

### Tests
- New tests: [count], covering: [list scenarios]
- All tests passing: [yes/no, with evidence]
- Coverage of new code: [estimate]

### Known Risks
- [risk 1]
- [risk 2]

### Verification Evidence
- Build: [exit code]
- Lint: [0 errors / N warnings]
- Tests: [X passed, Y skipped, 0 failed]
```

**Deliverables:** Code + Tests + Self-Assessment Report

---

### Role 2: Spec Reviewer (Requirements Verification)

**Core principle: DO NOT trust the Implementer's self-assessment. Verify independently.**

#### Checklist (every item must be evaluated)

- [ ] Every requirement has a corresponding implementation
- [ ] Every requirement has corresponding test coverage
- [ ] Boundary conditions and error paths are handled
- [ ] Interface contracts (params, return types, error codes) match specification
- [ ] No requirements were silently dropped or partially implemented

#### Output Format

```markdown
## Spec Review

| # | Requirement | Implementation | Test Coverage | Verdict |
|---|---|---|---|---|
| 1 | [requirement text] | [file:line] | [test file:line] | ✅ / ❌ |
| 2 | ... | ... | ... | ... |

### Issues (if any)
- ❌ REQ-2: [description of gap] — `src/auth.ts:42`
```

**Any ❌ blocks progression to Role 3.** Must fix and re-review.

---

### Role 3: Quality Reviewer (Code Quality & Security)

**Only starts AFTER Spec Reviewer gives all ✅.**

#### Checklist

- [ ] **Maintainability**: Clear naming, reasonable structure, acceptable complexity
- [ ] **Test quality**: Tests are meaningful (not just existence checks), cover edge cases
- [ ] **Code style**: Follows project conventions (not just linter rules)
- [ ] **Security**: No hardcoded secrets, injection risks, permission leaks, unsafe deserialization
- [ ] **Performance**: No obvious regressions (N+1 queries, unbounded loops, memory leaks)
- [ ] **Dependencies**: No unnecessary new dependencies; existing ones used correctly

#### Output Format

```markdown
## Quality Review

| # | Dimension | Verdict | Notes |
|---|---|---|---|
| 1 | Maintainability | ✅ / ⚠️ / ❌ | [details if not ✅] |
| 2 | Test quality | ✅ / ⚠️ / ❌ | ... |
| 3 | Code style | ✅ / ⚠️ / ❌ | ... |
| 4 | Security | ✅ / ⚠️ / ❌ | ... |
| 5 | Performance | ✅ / ⚠️ / ❌ | ... |
| 6 | Dependencies | ✅ / ⚠️ / ❌ | ... |

### Issues (if any)
- ❌ CRITICAL: [description] — `file:line`
- ⚠️ IMPORTANT: [description] — `file:line`
- 💡 SUGGESTION: [description] — `file:line`
```

---

## 4. Issue Severity & Handling

| Severity | Definition | Required Action |
| --- | --- | --- |
| ❌ Critical | Functional error, security vulnerability, data loss risk | Must fix → **restart full Three-Agent Pipeline** |
| ⚠️ Important | Poor maintainability, insufficient tests, performance risk | Must fix → **re-run Quality Reviewer only** |
| 💡 Suggestion | Style preference, optional optimization | Record but don't block delivery |

### Anti-Loop Rule

- Same issue: max **2 rounds** of fix-and-re-review
- After 2 rounds still unresolved → escalate to user for decision
- Re-review only covers modified code, not unchanged sections

---

## 5. Review Delegation Patterns

### For Spec Review (Role 2)

```typescript
task(
  subagent_type="oracle",
  load_skills=["agent-review-protocol"],
  run_in_background=false,
  description="Spec review: [feature name]",
  prompt=`
TASK: Spec Review (Role 2 of Three-Agent Review Pipeline)
EXPECTED OUTCOME: Per-requirement verdict table with ✅ or ❌ for each item.
REQUIRED TOOLS: Read, Grep, Glob (read-only — NO edits)
MUST DO:
- Independently verify each requirement has implementation AND test coverage
- Check interface contracts match specification
- Check boundary/error paths
- Output the exact table format from agent-review-protocol skill §3 Role 2
MUST NOT DO:
- Trust the implementer's self-assessment
- Skip any requirement
- Edit any files
CONTEXT:
- Requirements: [path to requirements/spec]
- Implementation: [paths to changed files]
- Tests: [paths to test files]
- Self-assessment: [paste or reference]
`
)
```

### For Quality Review (Role 3)

```typescript
task(
  subagent_type="oracle",
  load_skills=["agent-review-protocol"],
  run_in_background=false,
  description="Quality review: [feature name]",
  prompt=`
TASK: Quality Review (Role 3 of Three-Agent Review Pipeline)
EXPECTED OUTCOME: Quality dimension table with ✅/⚠️/❌ verdicts.
REQUIRED TOOLS: Read, Grep, Glob (read-only — NO edits)
MUST DO:
- Evaluate all 6 dimensions: maintainability, test quality, code style, security, performance, dependencies
- Flag any ❌ Critical issues that block delivery
- Reference specific file:line for all issues
- Output the exact table format from agent-review-protocol skill §3 Role 3
MUST NOT DO:
- Edit any files
- Mark ⚠️/❌ without specific file:line evidence
- Skip security check
CONTEXT:
- Spec Review passed: [confirmed]
- Implementation: [paths to changed files]
- Tests: [paths to test files]
- Project conventions: [reference AGENTS.md or style guide]
`
)
```

### For Simplified Review (Single Check)

```typescript
task(
  subagent_type="oracle",
  load_skills=[],
  run_in_background=false,
  description="Quick review: [change summary]",
  prompt=`
Review this small change for correctness and style consistency.
Files: [paths]
Change: [summary]
Return: PASS or specific issues with file:line references.
`
)
```

---

## 6. Integration with Workflow

### When in the Development Cycle

```
Plan → Plan Review (agent-workflow-rules §3) → Implement (TDD) → Self-Assessment → 
Three-Agent Review (this skill) → Fix issues → Re-review → Deliver
```

### Relationship to Other Gates

| Gate | Governed by | When |
| --- | --- | --- |
| Plan Review | `agent-workflow-rules` §3 | Before implementation starts |
| TDD Enforcement | `agent-workflow-rules` §2 | During implementation |
| Verification | `agent-workflow-rules` §4 | Before claiming done |
| Code Review | **This skill** | After implementation, before delivery |
| Cross-Check | **This skill** §1 | Always, for any completed work |

---

## 7. Review Evidence Requirements

A review is NOT complete without:

- [ ] Reviewer output with per-item verdicts
- [ ] All ❌ items resolved (with fix evidence)
- [ ] Final reviewer output showing all ✅
- [ ] Tests passing after any review-prompted fixes

**Forbidden:** Claiming "reviewed" without reviewer output artifact.
