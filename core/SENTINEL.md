---
name: Sentinel H1-H10
description: Spec quality self-audit — 10 checks to catch weak, misleading, or fake-green tests before execution.
version: 1.0.0
---

# Sentinel H1-H10 — Spec Quality Self-Audit

Scan EVERY test case before execution. If any check fails → fix BEFORE running.

## Checks

| ID | Principle | Detection | Correction |
|----|-----------|-----------|------------|
| H1 | **Mock-as-sole-assertion** — test only verifies a mock was called, not the actual behavior | [ADAPTER: assertions.mock_assertion_pattern] as the only assertion in a test | Add assertion on the actual response/result/side-effect |
| H2 | **Empty test body** — test has no assertion at all | Test case without any assertion statement | Add `expect`/`assert` for the behavior under test |
| H3 | **Orphan record reference** — test queries "last"/"first" record without creating it in the test | [ADAPTER: assertions.last_record_pattern] without corresponding creation | Create the record explicitly in the test setup |
| H4 | **Weak assertion as final check** — vague assertion when exact value is knowable | [ADAPTER: assertions.weak_assertions] as the final/only assertion | Replace with exact value assertion (e.g., `== 42` instead of `!= nil`) |
| H5 | **Missing invalid-input scenario** — parameter accepted without any negative test | Parameter used in the diff has no test with nil/invalid/empty input | Add test context with nil, invalid type, empty value |
| H6 | **Status-only assertion** — only checks HTTP status code, ignores response content | [ADAPTER: assertions.status_only_pattern] as the only assertion | Add assertion on response body, side-effects, or database state |
| H7 | **Mock of code-under-test** — mocking the very method/function being tested | [ADAPTER: assertions.mock_syntax] applied to a method from the current diff | Remove mock, exercise the real implementation |
| H8 | **Unhandled exception from input** — adversarial input causes unrescued crash | Test raises exception instead of returning error response | Register as Bug finding → Phase 4 |
| H9 | **Mock of auth/model in integration test** — faking auth or data in what should be a real integration | [ADAPTER: assertions.auth_mock_pattern] in an integration/endpoint test | Use real auth helpers and real data creation |
| H10 | **Mock of newly-created method** — mocking a method that was created or modified in this diff | [ADAPTER: assertions.mock_syntax] on a method that appears in `git diff` as added/changed | Remove mock, test the real new implementation |

## Exceptions

| Check | Acceptable Exception |
|-------|---------------------|
| H1 | Verifying async job was enqueued (mock + separate job spec) |
| H4 | `not null` / `not nil` as a guard before more specific assertions |
| H6 | Status + error message assertions combined |
| H7, H9, H10 | External dependencies without global stubs (SDKs, paid APIs, binaries) |

## How to Apply

1. After writing all specs (Phase 3.2), before executing (Phase 3.6)
2. Read each test case sequentially
3. For each test, run all 10 checks using adapter-provided patterns
4. Any violation → fix immediately
5. Log: `SENTINEL: N checks, N violations found, N fixed`

## Adapter Requirements

The adapter must provide these patterns in `assertions` section:

```yaml
assertions:
  mock_assertion_pattern: "pattern to detect mock-was-called assertions"
  last_record_pattern: "pattern to detect last/first record queries"
  weak_assertions:
    - "pattern1 for truthy/present/nil checks"
    - "pattern2..."
  status_only_pattern: "pattern to detect HTTP status-only assertions"
  mock_syntax: "pattern to detect mocking/stubbing"
  auth_mock_pattern: "pattern to detect mocked authentication"
```
