---
name: Adversarial Testing Methodology
description: How to systematically try to break code — payloads, 3-assertion pattern, invariants, state corruption, IDOR.
version: 1.0.0
---

# Adversarial Testing Methodology

The goal is to **try to break the code**, not to prove it works. Every adversarial test must verify 3 things.

## The 3-Assertion Pattern

Every adversarial test MUST verify:

```pseudocode
given: invalid/malicious input

1. NO UNHANDLED EXCEPTION — the action completes without crashing
2. FRIENDLY RESPONSE — status code is one of [200, 302, 400, 422] (not 500)
3. NO DATA CORRUPTION — no invalid data was persisted to the database
```

A test that only checks "no exception" is incomplete. A test that only checks status is incomplete. All 3 are required.

## Payload Table

For each new/changed parameter, use these adversarial inputs:

| Type | [MUST] test | [SHOULD] test |
|------|-------------|---------------|
| String | `null`, `""`, `"<script>alert(1)</script>"` | 10,000+ chars, emoji, unicode edge cases |
| Integer/Decimal | `null`, `""`, `"abc"`, `-1`, `0` | `999_999_999`, very small decimals |
| Date | `null`, `""`, `"not-a-date"` | `"31/02/2026"`, far past/future dates |
| ID (FK) | `null`, `0`, `999999` (non-existent) | — |
| Boolean | `null`, `""`, `"true"` (string vs boolean) | — |
| Enum/Status | `null`, `""`, `"INVALID_VALUE"` | value outside defined enum |
| File upload | `null`, empty file, wrong MIME type | oversized file, polyglot file |
| Array/List | `null`, `[]`, single item, 1000+ items | nested arrays |
| JSON/Object | `null`, `{}`, missing required keys | extra keys, deeply nested |

## Invariant Testing

[MUST for calculations/financial logic]

An invariant is a property that must ALWAYS hold true regardless of input. Examples:

```pseudocode
# Sum of parts equals total
assert sum(installments.amount) == invoice.total

# Balance never negative (if business rule)
assert account.balance >= 0

# Percentage between 0 and 100
assert 0 <= tax_rate <= 100

# Idempotency
result_1 = process(input)
result_2 = process(input)
assert result_1 == result_2
```

Max 2 invariant tests per file. Focus on the most critical business rules.

## State Corruption Testing

[SHOULD for update/delete operations]

Test what happens when state changes between operations:

```pseudocode
# Record deleted between read and update
record = create(valid_data)
delete(record)
response = update(record.id, new_data)
assert: no crash, friendly error, no ghost record created

# Concurrent modification
record = create(valid_data)
update_1 = update(record.id, data_a)  # succeeds
update_2 = update(record.id, data_b)  # should handle conflict
assert: data integrity maintained

# Double submission
response_1 = create(same_data)
response_2 = create(same_data)
assert: no duplicate, appropriate error on second attempt
```

## IDOR (Insecure Direct Object Reference)

[MUST for every parameter-based record lookup in endpoints]

IDOR occurs when a user can access/modify resources belonging to another user/tenant by manipulating IDs in request parameters.

### Detection

[ADAPTER: idor_detection.scan_command] — scan endpoints for parameter-based lookups.

- **SAFE**: lookup scoped to current user/tenant
- **VULNERABLE**: global/unscoped lookup

### Test Pattern

```pseudocode
context "IDOR: resource from another tenant"
  setup:
    current_user = create_user(tenant_A)
    other_user = create_user(tenant_B)
    foreign_resource = create_resource(owner: other_user)
    authenticate_as(current_user)

  test "cannot access foreign resource":
    response = GET /resource/{foreign_resource.id}
    assert response.status in [302, 403, 404]

  test "cannot modify foreign resource":
    response = PUT /resource/{foreign_resource.id}, data: {field: "hacked"}
    assert response.status in [302, 403, 404]
    assert foreign_resource.reload.field != "hacked"

  test "cannot delete foreign resource":
    response = DELETE /resource/{foreign_resource.id}
    assert resource still exists in database

  test "cannot associate foreign resource":
    response = POST /parent, data: {child_id: foreign_resource.id}
    assert no record created with foreign_resource association
```

### IDOR Self-Audit

Before executing IDOR specs, verify:
1. Every `params[:*_id]` / route parameter has a cross-tenant spec
2. The "other tenant" is a completely separate user, not just a different role
3. Assertions check BOTH access denial AND data integrity (not just status code)

## When to Apply

| PR_TYPE | Adversarial Level |
|---------|------------------|
| SECURITY | FULL — all payload types, all IDOR, invariants |
| LOGIC | STANDARD — key payloads, IDOR if params, invariants for calculations |
| ENDPOINT | STANDARD — key payloads, IDOR if params |
| VIEW | INPUT_ONLY — string payloads (XSS), boundary values |
| CONFIG | NONE — smoke tests only |
