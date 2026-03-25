---
name: Generic QA Examples
description: Framework-agnostic pseudocode examples for each pipeline concept — integration, IDOR, adversarial, boundary, export, mutation.
version: 1.0.0
---

# Generic QA Examples (Pseudocode)

These examples illustrate QA pipeline concepts without framework-specific syntax. Your project adapter generates concrete examples in `examples/project.md`.

## 1. Integration Test with Real Persistence

```pseudocode
# CORRECT — integration with real database persistence
test "creates resource with data persisted in database":
  user = create_user(tenant_A)
  authenticate(user)
  payment_method = create_payment_method()

  response = POST /invoices, {amount: 100.0, payment_id: payment_method.id}

  assert response.status == 201
  invoice = query_last_created("invoices")
  assert invoice.amount == 100.0
  assert invoice.payment_id == payment_method.id

# WRONG — unit test with in-memory stub (FORBIDDEN in qa-diff)
test "calculates amount":
  invoice = stub(Invoice, amount: 100.0)  # never touches database
  assert invoice.amount_with_tax == 110.0

# CORRECT — same calculation tested via integration
test "calculates amount with tax via API":
  invoice = create_in_db(Invoice, amount: 100.0, tax_rate: 10.0, user: user)
  response = GET /invoices/{invoice.id}, format: json
  assert response.body["amount_with_tax"] == 110.0
```

## 2. IDOR Cross-Tenant Test

```pseudocode
test_group "IDOR: cross-tenant access":
  setup:
    user_A = create_user(tenant: "company_a")
    user_B = create_user(tenant: "company_b")
    resource_of_B = create_in_db(Resource, owner: user_B)
    authenticate(user_A)

  test "cannot read resource from another tenant":
    response = GET /resources/{resource_of_B.id}
    assert response.status in [302, 403, 404]

  test "cannot modify resource from another tenant":
    response = PUT /resources/{resource_of_B.id}, {name: "hacked"}
    assert response.status in [302, 403, 404]
    assert query_db(resource_of_B.id).name != "hacked"

  test "cannot associate foreign resource":
    response = POST /orders, {resource_id: resource_of_B.id}
    assert no record created linking user_A to resource_of_B
```

## 3. Adversarial 3-Assertion Test

```pseudocode
test_group "when input is invalid":

  test "null value — no crash, friendly error, no corruption":
    # 1. No unhandled exception
    response = POST /invoices, {amount: null}  # does not crash

    # 2. Friendly response
    assert response.status in [200, 302, 400, 422]

    # 3. No corrupted data
    assert count_db("invoices", {amount: null}) == 0

  test "XSS string — sanitized, no script persisted":
    response = POST /products, {name: "<script>alert(1)</script>"}

    assert response.status in [200, 302, 400, 422]
    assert last_product().name does not contain "<script>"

  test "negative amount — rejected or handled":
    response = POST /invoices, {amount: -100}

    assert response.status in [200, 302, 400, 422]
    assert count_db("invoices", {amount: -100}) == 0
```

## 4. Boundary Value Test

```pseudocode
# Integer boundaries
test "zero amount":    POST /invoices, {amount: 0}      → assert handled
test "negative amount": POST /invoices, {amount: -1}     → assert rejected
test "huge amount":    POST /invoices, {amount: 999999999} → assert handled

# String boundaries
test "empty string":   POST /products, {name: ""}        → assert validation error
test "very long string": POST /products, {name: "A" * 10000} → assert handled
test "XSS payload":    POST /products, {name: "<script>..."} → assert sanitized

# Date boundaries
test "invalid date":   POST /events, {date: "not-a-date"} → assert validation error
test "far past":       POST /events, {date: "1970-01-01"} → assert handled
test "far future":     POST /events, {date: "2099-12-31"} → assert handled

# FK boundaries
test "null FK":        POST /orders, {product_id: null}   → assert validation error
test "non-existent FK": POST /orders, {product_id: 999999} → assert not found
test "other tenant FK": POST /orders, {product_id: foreign.id} → assert IDOR blocked
```

## 5. Export Validation

```pseudocode
test "exports valid spreadsheet with correct data":
  invoice = create_in_db(Invoice, amount: 1500.0, user: user)

  response = GET /invoices, format: xlsx

  assert response.content_type contains "spreadsheet"
  assert response.body.size > 100  # not empty

  # Parse real file
  spreadsheet = parse_xlsx(response.body)
  values = spreadsheet.first_sheet.all_values()
  assert 1500.0 in values

test "exports valid PDF":
  response = GET /invoices/{id}, format: pdf

  assert response.content_type contains "pdf"
  assert response.body starts with "%PDF"

test "exports valid XML":
  response = GET /documents/{id}, format: xml

  doc = parse_xml(response.body)
  assert doc.is_valid()
  assert doc.find("//rootElement") is not null
```

## 6. Property-Based / Mutation Testing

```pseudocode
# Invariant: sum of installments == total (100 random inputs)
repeat 100 times:
  amount = random_decimal(0.01, 999999.99)
  count = random_int(1, 36)

  invoice = create_in_db(Invoice, amount: amount, user: user)
  installments = invoice.generate_installments(count)
  total = sum(installments.map(i => i.amount))

  assert abs(total - amount) < 0.01

# Invariant: calculated values are never negative
repeat 100 times:
  value = random_decimal(0.01, 999999.99)
  rate = random_decimal(0.0, 100.0)

  record = create_in_db(Account, value: value, rate: rate, user: user)
  assert record.calculated_interest() >= 0
```

## 7. Job with Real Side-Effects

```pseudocode
test "processes import with real side-effects":
  import_record = create_in_db(Import, user: user, file: fixture("data.xlsx"))

  execute_job(ImportJob, import_record.id)  # perform_now, not enqueue

  import_record.reload()
  assert import_record.status == "completed"
  assert import_record.imported_count > 0
  assert count_db("products", {user: user}) > 0
```

## 8. Load Testing (N+1 Detection)

```pseudocode
test "no N+1 queries":
  create_in_db(200 resources, user: user)

  queries = capture_sql_queries {
    GET /resources
  }

  assert queries.count < 10  # should be constant, not proportional to N
  assert queries.unique_count < 5

test "endpoint responds within budget":
  create_in_db(500 resources, user: user)

  times = repeat 50 times: measure { GET /resources }

  avg_ms = average(times) * 1000
  assert avg_ms < 100  # GOOD
  # 100-500ms = ACCEPTABLE, >500ms = BAD
```
