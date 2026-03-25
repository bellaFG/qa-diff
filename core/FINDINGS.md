---
name: Finding Classification
description: How to classify, score, and triage QA findings — types, severity, FRAGILE subtypes, verdict logic.
version: 1.0.0
---

# Finding Classification

## Finding Types

| Type | Description | Scope |
|------|-------------|-------|
| **Bug** | Code in the PR has wrong behavior (crash, wrong result, missing validation) | PR |
| **Vulnerability** | Security issue: IDOR, SQL injection, XSS, mass assignment, etc. | PR |
| **Improvement** | Code works but could be better (readability, edge case, performance) | PR or tech debt |
| **FRAGILE** | Works today but will break under specific conditions | Adjacent code |

## Severity

| Level | Meaning | Impact on Verdict |
|-------|---------|-------------------|
| **CRITICAL** | Data loss, security breach, financial error, crash in happy path | BLOCKED (if Bug/Vulnerability) |
| **HIGH** | Wrong behavior in common edge case, missing validation for common input | Must be triaged |
| **MEDIUM** | Wrong behavior in rare edge case, suboptimal error handling | Should be triaged |
| **LOW** | Minor improvement, cosmetic, unlikely edge case | May be deferred |

## FRAGILE Subtypes

| Subtype | Description | Example |
|---------|-------------|---------|
| Implicit ordering | Query relies on DB default order without explicit ORDER BY | `Model.last` without `.order()` |
| Null assumption | Code assumes non-null but column/field is nullable | `record.field.length` without null check |
| Broad exception catch | Catches too-wide exception class without re-raising | `rescue Exception` / `except Exception` |
| Raw SQL without binding | SQL string interpolation instead of parameterized query | `"WHERE id = #{id}"` |
| Magic string/number | Business logic depends on hardcoded string comparison | `if status == "F"` |
| Missing index | Query on column without database index (performance time bomb) | `WHERE foreign_key = ?` on unindexed column |
| Race condition | Non-atomic read-then-write without locking | Check-then-act without transaction |

## Failure Classification (Phase 3.9)

When a spec fails, classify the failure:

| Type | Description | Action |
|------|-------------|--------|
| **A** | Spec is incorrect (wrong assertion, wrong setup) | QA fixes its own spec |
| **B** | Real bug in production code | Register as Bug finding → Phase 7 |
| **C** | Missing test setup (fixture, seed data, env var) | QA fixes its own spec |
| **D** | Environment issue (DB down, service unavailable) | Register as Blocker |
| **E** | Input causes unhandled exception | Register as Bug finding → Phase 7 |
| **F** | IDOR or security vulnerability confirmed | Register as Vulnerability finding → Phase 7 |

## Verdict Logic

| Verdict | Condition |
|---------|-----------|
| **APPROVED** | Zero open findings after triage |
| **APPROVED_WITH_PENDING** | Findings exist but dev chose to ignore/defer them in triage. QA obligation fulfilled |
| **BLOCKED** | Environment issues prevent testing, OR >50% of files could not be tested |

## Triage Workflow

Each finding gets one of 3 dispositions:

| Disposition | Code | Meaning |
|-------------|------|---------|
| Coder fixes | `C` | Finding context passed to coder for immediate fix |
| New card | `I` | Issue created with finding context for later fix |
| Ignore | `X` | Decision recorded in report; finding becomes known risk |

Findings marked `X` (ignored) → record as acknowledged risk in the report.
Findings that are consistently false positives → record in project learnings for future calibration.
