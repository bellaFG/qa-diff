---
name: QA Diff Onboarding
description: Interactive skill that detects the project stack, asks clarifying questions, and generates adapter + examples + config for the QA pipeline.
version: 1.0.0
---

# QA Diff — Onboarding (`/qa-setup`)

You are the setup wizard for the QA Diff pipeline. Your job is to detect the project's stack, ask the developer a few questions, and generate the configuration files that make `/qa-diff` work.

## Output Files

By the end of this process, you will generate:

1. `adapter/project.md` — project-specific adapter (satisfies `adapter/INTERFACE.md`)
2. `examples/project.md` — concrete code examples for this stack
3. `config.yml` — project paths and settings
4. (Optional) `.github/workflows/qa-gate.yml` — CI gate

## Process

### Step 1 — Auto-Detect Stack

Scan the project root for these indicators:

```
| File/Dir | Language | Framework |
|----------|----------|-----------|
| Gemfile, Gemfile.lock | Ruby | check for 'rails' |
| package.json | JavaScript/TypeScript | check for next, express, vue, react, nuxt, svelte |
| requirements.txt, pyproject.toml, Pipfile | Python | check for django, fastapi, flask, starlette |
| go.mod | Go | check for gin, echo, fiber, chi |
| Cargo.toml | Rust | check for actix, axum, rocket |
| pom.xml, build.gradle, build.gradle.kts | Java/Kotlin | check for spring, quarkus, micronaut |
| mix.exs | Elixir | check for phoenix |
| composer.json | PHP | check for laravel, symfony |
```

Also detect test runner:

```
| Indicator | Test Runner |
|-----------|-------------|
| spec/ dir + Gemfile has 'rspec' | RSpec |
| test/ dir + Gemfile has 'minitest' | Minitest |
| pytest.ini, conftest.py, pyproject.toml [tool.pytest] | pytest |
| test/ dir + Python without pytest | unittest |
| jest.config.*, package.json has 'jest' | Jest |
| vitest.config.*, package.json has 'vitest' | Vitest |
| *_test.go files | go test |
| tests/ dir + Cargo.toml | cargo test |
```

Also detect coverage tool:

```
| Indicator | Coverage Tool |
|-----------|--------------|
| simplecov in Gemfile | SimpleCov |
| coverage in requirements/pyproject | coverage.py / pytest-cov |
| jest --coverage / c8 / istanbul | Istanbul/c8 |
| go test -cover | go cover |
```

Also detect security tools:

```
| Indicator | Security Tool |
|-----------|--------------|
| brakeman in Gemfile | Brakeman |
| bandit in requirements | Bandit |
| eslint-plugin-security | ESLint Security |
| gosec | GoSec |
| cargo-audit | cargo audit |
```

### Step 2 — Confirm Detection & Ask Questions

Present detected stack and ask for confirmation + clarifications via `AskUserQuestion`.

**Question 1 — Stack Confirmation:**
```
Detected: [Language] + [Framework] + [Test Runner]

Is this correct?
[yes] confirm · [no] let me specify
```

**Question 2 — Authentication Model:**
```
How does authentication work in your tests?

[session] Session-based (login form/helper that sets session)
[token] Token-based (JWT/API key in headers)
[none] No auth (public API)
[custom] Let me describe...
```

For each choice, ask for the specific helper/pattern:
- session: "What helper logs in a user in tests? (e.g., `login(user)`, `sign_in(user)`)"
- token: "How do you generate auth headers? (e.g., `auth_headers(user)`)"

**Question 3 — Multi-Tenancy:**
```
Does your app have multi-tenancy (user/org data isolation)?

[user_fk] Yes — records belong to a user via FK (user_id, account_id, etc.)
[org] Yes — records belong to an organization/workspace
[schema] Yes — separate DB schema per tenant
[none] No — single-tenant app
```

If yes:
- "What FK/field scopes records to a tenant? (e.g., `user_id`, `organization_id`)"
- "How do you create a second tenant in tests? (e.g., `create(:user, org: other_org)`)"

**Question 4 — Test Data Creation:**
```
How do you create test data?

[factory] Factories (FactoryBot, factory_boy, fishery, etc.)
[fixtures] Fixtures (YAML/JSON files)
[builder] Builder/helper functions
[inline] Inline creation (direct DB insert)
```

**Question 5 — CI Platform:**
```
Do you want to set up CI gates?

[github] GitHub Actions (recommended)
[gitlab] GitLab CI
[none] Skip CI setup for now
```

### Step 3 — Generate Files

Based on the answers, generate the three files:

#### 3.1 Generate `adapter/project.md`

Read `adapter/INTERFACE.md` for the contract. Fill in every REQUIRED section based on detected stack + answers. Fill OPTIONAL sections if the tools were detected.

**Key mappings by framework:**

**Rails:**
- file_patterns: `app/models/`, `app/controllers/`, `app/views/`, etc.
- auth: `login(user, pessoa)` or custom
- idor: `grep 'params\[:.*_id\]'`, safe = `current_user.*.find()`, vulnerable = `Model.find()`
- test: `bundle exec rspec`, `_spec.rb`, `spec/`
- assertions: `have_received`, `be_present`, `have_http_status`, `allow().to receive`

**Django:**
- file_patterns: `*/models.py`, `*/views.py`, `*/serializers.py`, `*/templates/`
- auth: `self.client.force_authenticate(user)` or `self.client.login()`
- idor: `grep 'kwargs\["pk"\]'` or `request.data.get("id")`, safe = `queryset.filter(owner=request.user)`, vulnerable = `Model.objects.get(pk=pk)`
- test: `pytest` or `python manage.py test`, `test_*.py`
- assertions: `mock.assert_called`, `assertIsNotNone`, `assertEqual`

**Express/Next.js:**
- file_patterns: `src/routes/`, `src/controllers/`, `src/models/`, `pages/api/`
- auth: `request(app).set('Authorization', token)`
- idor: `req.params.id` without `where: { userId }`, vulnerable = `Model.findByPk(id)`
- test: `jest` or `vitest`, `.test.ts`, `__tests__/`
- assertions: `toHaveBeenCalled`, `toBeTruthy`, `toEqual`

**Go:**
- file_patterns: `internal/handlers/`, `internal/models/`, `internal/services/`
- auth: middleware mock or test JWT
- idor: `c.Param("id")` without user scoping
- test: `go test ./...`, `_test.go`
- assertions: `assert.Equal`, `assert.Nil`, `require.NoError`

**FastAPI:**
- file_patterns: `app/routers/`, `app/models/`, `app/schemas/`
- auth: `client.headers = {"Authorization": f"Bearer {token}"}`
- idor: `path parameter id` without `filter(owner_id=current_user.id)`
- test: `pytest`, `test_*.py`
- assertions: `mock.assert_called`, `assert response.json()["field"] == value`

For frameworks not listed: use the detected patterns and ask the developer for specifics.

#### 3.2 Generate `examples/project.md`

Write 6-8 concrete code examples using the project's actual syntax:
1. Integration test with real persistence
2. IDOR cross-tenant test
3. Adversarial 3-assertion test
4. Boundary value test
5. Export validation (if applicable)
6. Job/worker test (if applicable)
7. Property-based/mutation test (if calculations exist)
8. Load test template (if performance-relevant)

Use `examples/generic.md` as the conceptual reference, translate to the project's test syntax.

#### 3.3 Generate `config.yml`

```yaml
# Generated by /qa-setup on YYYY-MM-DD
stack:
  language: <detected>
  framework: <detected>
  test_runner: <detected>

paths:
  report_dir: qa-reports    # where reports are saved (committed to repo)
  temp_dir: /tmp/qa-specs   # where temp specs live (never committed)
  source_dirs: "<detected source directories>"

auth:
  model: <session|token|none>
  login_helper: "<from question 2>"
  tenant_field: "<from question 3>"
  other_tenant_helper: "<from question 3>"

ci:
  platform: <github|gitlab|none>
  target_branches: [main]
```

#### 3.4 (Optional) Install CI

If dev chose GitHub Actions:
1. Read `ci/qa-gate.yml` template
2. Replace placeholders with values from `config.yml`
3. Write to `.github/workflows/qa-gate.yml`
4. Same for `ci/qa-cleanup.yml`

If dev chose GitLab CI:
1. Generate equivalent `.gitlab-ci.yml` stage

### Step 4 — Verify & Report

After generating files:

1. Read `adapter/INTERFACE.md` and validate that `adapter/project.md` satisfies all REQUIRED sections
2. List generated files with a brief description
3. Show the developer how to run their first QA:

```
Setup complete! Generated files:
  - adapter/project.md (your project's QA adapter)
  - examples/project.md (concrete test examples for your stack)
  - config.yml (project configuration)
  - .github/workflows/qa-gate.yml (CI gate) [if applicable]

Run your first QA:
  /qa-diff

Or specify a PR:
  /qa-diff 123
```

## Edge Cases

- **Monorepo**: If multiple `package.json` / `Gemfile` detected at different paths, ask which one is the primary app.
- **Multiple languages**: If both `Gemfile` and `package.json` exist (e.g., Rails + frontend), detect the primary backend and note the frontend. Generate adapter for the backend; frontend testing is noted as optional in adapter.
- **No test runner detected**: Ask the developer what they use. If none, warn that qa-diff requires a test runner and suggest setting one up first.
- **Custom frameworks**: If no known framework detected, ask for file layout and test patterns. Generate a minimal adapter with what's known.
