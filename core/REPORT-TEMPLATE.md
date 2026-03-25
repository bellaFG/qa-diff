---
name: Report Template
description: Standard format for QA diff reports — frontmatter schema, sections, and examples.
version: 1.0.0
---

# QA Report Template

## Frontmatter (YAML)

Every report MUST include this frontmatter:

```yaml
---
artifact: qa-diff-report
branch: <branch name>
date: YYYY-MM-DD
head_sha: <output of `git rev-parse HEAD` at execution time>
verdict: APPROVED | APPROVED_WITH_PENDING | BLOCKED
pr_type: SECURITY | LOGIC | ENDPOINT | VIEW | CONFIG
pr_size: Micro | Small | Medium | Large
idor_detected: true | false
mode: full | incremental
---
```

### Field Definitions

| Field | Required | Description |
|-------|----------|-------------|
| `artifact` | yes | Always `qa-diff-report` |
| `branch` | yes | Branch name (not sanitized) |
| `date` | yes | ISO date of execution |
| `head_sha` | yes | Full SHA of HEAD at execution time. CI uses this to detect code changes after QA |
| `verdict` | yes | Final verdict (see core/FINDINGS.md for logic) |
| `pr_type` | yes | Highest-priority type detected |
| `pr_size` | yes | Based on file count in source directories |
| `idor_detected` | yes | Whether unscoped parameter lookup was found |
| `mode` | yes | `full` (first run) or `incremental` (re-run after fixes) |

## Report Sections

```markdown
## QA REPORT — [VERDICT]

[branch] · [date] · TYPE: [X] · SIZE: [Y] · Budget: [used]/[total] min · Mode: [full/incremental]

### Summary
Specs: N total · N failures · ALL in /tmp/
Coverage: N% · IDOR: N · Bugs: N · FRAGILE: N
H1-H10: [checkmarks per check]
Mutation: score=N% (if applicable)

### IDOR Audit
| Param | Location | Pattern | Status |
|-------|----------|---------|--------|
| :id | file.ext:42 | unscoped lookup | VULNERABLE |
| :parent_id | file.ext:78 | scoped via current_user | SAFE |

### Coverage
| File | % diff lines | Uncovered lines |
|------|-------------|-----------------|
| src/handler.ext | 95.2% | 42, 67 |

### Adversarial
| Payload | Target | Result |
|---------|--------|--------|
| null supplier_id | POST /orders | 422 + no corruption |
| XSS in name | POST /products | sanitized, no script persisted |

### Mutation Testing (if applicable)
| File | Mutants | Killed | Alive | Score |
|------|---------|--------|-------|-------|
| calculator.ext | 20 | 18 | 2 | 90% |

### Load Testing (if KR)
| Metric | Value | Status |
|--------|-------|--------|
| Avg | 85ms | GOOD |
| P95 | 150ms | GOOD |
| N+1 | No | GOOD |

### Findings (if any)
**Finding 1** · [file:line] · Bug · CRITICAL · [CODER/CARD/IGNORED]
Description: ...
Actual: ...
Expected: ...

**Finding 2** · [file:line] · FRAGILE · MEDIUM · [CODER/CARD/IGNORED]
Description: ...

### Specs created (temporary)
- `/tmp/qa-specs/endpoints/orders_spec.ext` (5 examples, 0 failures)
- `/tmp/qa-specs/endpoints/products_spec.ext` (3 examples, 1 failure)

### Next steps
[Populated by the adapter with project-specific commands]
```

## File Naming

Report file: `{report_dir}/qa-report-{BRANCH_SAFE}.md`

Where `BRANCH_SAFE` = branch name with `/` replaced by `-`.

Example: branch `feature/add-payments` → `qa-report-feature-add-payments.md`

## CI Integration

The `head_sha` field enables staleness detection:
1. CI extracts `head_sha` from the report frontmatter
2. CI compares it with the latest relevant commit SHA (excluding report commits)
3. If they differ → code changed after QA → fail the gate
