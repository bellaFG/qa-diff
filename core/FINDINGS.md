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

Applies to ALL finding types including FRAGILE:

| Level | Meaning | Impact on Verdict |
|-------|---------|-------------------|
| **CRITICAL** | Data loss, security breach, financial error, crash in happy path | BLOCKED (if Bug/Vulnerability). Must be triaged (if FRAGILE/Improvement) |
| **HIGH** | Wrong behavior in common edge case, missing validation for common input | Must be triaged |
| **MEDIUM** | Wrong behavior in rare edge case, suboptimal error handling | Should be triaged |
| **LOW** | Minor improvement, cosmetic, unlikely edge case | May be deferred |

### FRAGILE and Verdict

FRAGILE findings are in **adjacent code** (not the PR itself), so they follow special rules:

- FRAGILE **never blocks alone** — it's not code the developer changed
- FRAGILE-CRITICAL (race condition, raw SQL injection, null assumption on financial data) → **must appear in triage** (Phase 7), developer decides
- FRAGILE-HIGH/MEDIUM/LOW → included in report, appears in triage if other findings exist
- If ALL findings are FRAGILE and developer ignores them → verdict = `APPROVED_WITH_PENDING`

## FRAGILE Subtypes

| Subtype | Default Severity | Example |
|---------|-----------------|---------|
| Raw SQL without binding | CRITICAL | `"WHERE id = #{id}"` — SQL injection risk |
| Race condition | CRITICAL | Check-then-act without transaction |
| Null assumption | HIGH | `record.field.length` without null check on nullable column |
| Implicit ordering | MEDIUM | `Model.last` without `.order()` |
| Broad exception catch | MEDIUM | `rescue Exception` / `except Exception` |
| Magic string/number | LOW | `if status == "F"` |
| Missing index | LOW | `WHERE foreign_key = ?` on unindexed column |

## Failure Classification (Phase 3.9)

When a spec fails, classify the failure:

| Type | Description | Action |
|------|-------------|--------|
| **A** | Spec is incorrect (wrong assertion, wrong setup) | QA fixes its own spec |
| **B** | Real bug in production code | Register as Bug finding → Phase 7 |
| **C** | Missing test setup (fixture, seed data, env var) | QA fixes its own spec |
| **D** | Environment issue (DB down, service unavailable) | Register as Blocker |
| **E** | Input causes unhandled exception | Register as Bug finding (severity HIGH) → Phase 7 |
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
