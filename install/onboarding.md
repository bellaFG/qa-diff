---
name: QA Diff Onboarding
description: Interactive skill that deeply inspects the project repo, auto-detects the full stack, asks minimal clarifying questions, and generates adapter + examples + config for the QA pipeline.
version: 2.0.0
---

# QA Diff — Onboarding (`/qa-setup`)

You are the setup wizard for the QA Diff pipeline. Your job is to **deeply inspect the target repository**, auto-detect as much as possible, ask the developer only what can't be inferred, and generate the configuration files that make `/qa-diff` work.

## Output Files

By the end of this process, you will generate:

1. `adapter/project.md` — project-specific adapter (satisfies `adapter/INTERFACE.md`)
2. `examples/project.md` — concrete code examples for this stack
3. `config.yml` — project paths and settings
4. (Optional) `.github/workflows/qa-gate.yml` — CI gate

## Process

### Step 1 — Deep Auto-Detection

[MUST] Run the full detection from `install/detect-stack.md`. This means **actually reading files**, not just checking existence.

#### 1.1 Language & Package Manager

Check for dependency files (`Gemfile`, `package.json`, `requirements.txt`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `pom.xml`, `build.gradle`, `mix.exs`, `composer.json`, etc.).

Extract language version from `.ruby-version`, `.nvmrc`, `.node-version`, `.python-version`, `go.mod`, `rust-toolchain.toml`, etc.

#### 1.2 Framework — Read Dependency Files

[MUST] **Actually read** the dependency file contents — don't just check for file existence.

- **Ruby**: `cat Gemfile` → grep for `rails`, `sinatra`, `grape`, etc. Read `Gemfile.lock` for exact versions.
- **JS/TS**: `cat package.json` → parse `dependencies` and `devDependencies` for `next`, `express`, `fastify`, `nestjs`, `vue`, `react`, `svelte`, `angular`, etc. Also detect ORM (`prisma`, `drizzle`, `typeorm`, `sequelize`, `mongoose`).
- **Python**: `cat requirements.txt` or `cat pyproject.toml` → grep for `django`, `flask`, `fastapi`, `starlette`, etc. Also detect ORM (`sqlalchemy`, `django` built-in, `tortoise-orm`).
- **Go**: `cat go.mod` → grep for `gin`, `echo`, `fiber`, `chi`, etc.
- **Rust**: `cat Cargo.toml` → grep for `actix-web`, `axum`, `rocket`, etc.
- **Java/Kotlin**: `cat pom.xml` or `cat build.gradle` → grep for `spring-boot`, `quarkus`, `micronaut`, `ktor`, etc.
- **Elixir**: `cat mix.exs` → grep for `:phoenix`, `:plug`, `:absinthe`.
- **PHP**: `cat composer.json` → grep for `laravel`, `symfony`, `slim`, etc.

#### 1.3 Directory Structure

[MUST] Scan the filesystem to map actual source directories to pipeline categories:

```bash
find . -maxdepth 3 -type d -not -path './.git/*' -not -path './node_modules/*' -not -path './vendor/*' -not -path './.bundle/*' -not -path './tmp/*' -not -path './target/*' -not -path './dist/*' -not -path './build/*' -not -path './.next/*' 2>/dev/null
```

Map found directories to `file_patterns` using the framework-specific patterns from `detect-stack.md`.

#### 1.4 Test Runner & Config

[MUST] Detect test runner from:
- Dependency files (gems, npm packages, pip packages)
- Config files (`jest.config.*`, `vitest.config.*`, `.rspec`, `pytest.ini`, `conftest.py`, `phpunit.xml`)
- Test directory existence (`spec/`, `test/`, `tests/`, `__tests__/`)
- Test file naming patterns (`*_spec.rb`, `*.test.ts`, `test_*.py`, `*_test.go`)

Also detect E2E test runners (`cypress`, `playwright`, `capybara`, `selenium`).

#### 1.5 Coverage, Security, Linting

[MUST] Detect from dependency files and config files:
- **Coverage**: `simplecov`, `istanbul/c8/nyc`, `pytest-cov`, `jacoco`, `tarpaulin`
- **Security**: `brakeman`, `bandit`, `gosec`, `eslint-security`, `semgrep`, `snyk`, `bundler-audit`, `cargo-audit`
- **Linter**: `rubocop`, `eslint`, `biome`, `ruff`, `flake8`, `golangci-lint`, `clippy`, `credo`
- **Formatter**: `prettier`, `black`, `rustfmt`, `gofmt`
- **Type checker**: `mypy`, `pyright`, `typescript`, `phpstan`

#### 1.6 Database & ORM

Detect from:
- Database config files (`config/database.yml`, `settings.py`, `.env.example`)
- ORM dependencies in package manager files
- Docker-compose services (`postgres`, `mysql`, `mongo`, `redis`)

#### 1.7 Auth Library & Strategy

[MUST] Detect auth libraries from dependencies:
- Ruby: `devise`, `omniauth`, `jwt`, `cancancan`, `pundit`
- JS/TS: `passport`, `next-auth`, `lucia`, `clerk`, `jsonwebtoken`, `jose`
- Python: `django-allauth`, `flask-login`, `fastapi-users`, `python-jose`
- Go: `golang-jwt/jwt`, `go-oidc`
- PHP: Laravel Sanctum/Passport, `tymon/jwt-auth`

#### 1.8 Factory / Test Data Pattern

Detect factory libraries and directories:
- Ruby: `factory_bot`, `spec/factories/`
- JS/TS: `fishery`, `@faker-js/faker`, factory files
- Python: `factory_boy`, `pytest-factoryboy`, `faker`
- PHP: Laravel factories, `fakerphp/faker`
- Elixir: `ex_machina`

#### 1.9 Read Existing Test Files

[MUST] **Read 2-3 existing test files** to extract actual patterns:

```bash
# Find the most recent test files
find ${TEST_DIR} -name "${TEST_FILE_PATTERN}" -type f 2>/dev/null | xargs ls -t 2>/dev/null | head -3
```

From these files, extract:
1. **Auth helper** — exact function call used to authenticate in tests
2. **Factory usage** — exact syntax for creating test data
3. **Assertion style** — assertion patterns and libraries
4. **Setup/teardown** — how tests prepare state
5. **Tenant/user creation** — how tests create users and tenants

This is critical for generating adapters with the **exact patterns** the project already uses.

#### 1.10 API Style & CI/CD

- **API**: REST (default), GraphQL (`.graphql` files, `graphql` deps), gRPC (`.proto` files)
- **CI**: GitHub Actions (`.github/workflows/`), GitLab CI (`.gitlab-ci.yml`), CircleCI, Jenkins, etc.
- **Deploy**: Vercel, Netlify, Heroku, Fly.io, AWS, etc.
- **Container**: Dockerfile, docker-compose

#### 1.11 Monorepo Detection

Check for workspaces (`package.json` workspaces, `pnpm-workspace.yaml`, `turbo.json`, `nx.json`, `lerna.json`, `go.work`). If monorepo detected, identify which app/package is the primary target.

### Step 2 — Present Detection & Ask Minimal Questions

Present the full detection summary and ask **only what couldn't be auto-detected**.

**Always present — Detection Summary:**
```
Detected Stack:
  Language:       [language] [version]
  Framework:      [framework] [version]
  ORM/DB:         [orm] + [database]
  Test Runner:    [test_runner] (dir: [test_dir])
  Coverage:       [coverage_tool or "not detected"]
  Security:       [list or "none detected"]
  Linter:         [linter] + [formatter]
  Auth:           [auth_lib] ([strategy])
  Factories:      [factory_lib] (dir: [factory_dir])
  API:            [api_style]
  CI:             [ci_platform]

Auth Helper (from tests): [extracted pattern or "could not detect"]
Factory Pattern (from tests): [extracted pattern or "could not detect"]
Tenant Pattern (from tests): [extracted pattern or "could not detect"]

Is this correct? [yes] confirm · [no] let me adjust
```

**Conditional questions — only ask if NOT auto-detected:**

| Question | Ask when |
|----------|----------|
| Auth test helper | Auth library detected but exact test helper pattern NOT found in existing tests |
| Auth model (session/token) | No auth library detected at all |
| Multi-tenancy model | Always — business logic, not detectable from code |
| Tenant FK field | Only if multi-tenant (from Q above) |
| Test data creation | No factory library detected AND no pattern found in existing tests |
| Report language | Always — user preference |
| CI setup | CI platform NOT detected |

**Question flow for undetected items:**

If auth helper was auto-detected from reading tests → skip auth questions entirely.
If factory pattern was auto-detected → skip factory questions entirely.
If CI was auto-detected → only ask "Do you want QA gates added to your existing [platform] CI?"

**Multi-Tenancy (always asked — business logic):**
```
Does your app isolate data per user/organization (multi-tenancy)?

[user_fk] Yes — records belong to a user via FK (user_id, account_id, etc.)
[org] Yes — records belong to an organization/workspace
[schema] Yes — separate DB schema per tenant
[none] No — single-tenant app
```

If yes:
- "What FK/field scopes records to a tenant? (e.g., `user_id`, `organization_id`)"
- "How do you create a second tenant in tests? (e.g., `create(:user, org: other_org)`)"

**Report Language (always asked):**
```
What language should QA reports be written in?

[en] English · [pt] Portuguese (pt-BR) · [es] Spanish
```

### Step 3 — Generate Files

Based on auto-detection + answers, generate the three files:

#### 3.1 Generate `adapter/project.md`

Read `adapter/INTERFACE.md` for the contract. Fill in every REQUIRED section using:

1. **Auto-detected data** for file_patterns, test_execution, assertions, coverage, security_tools, lint
2. **Extracted patterns from existing tests** for auth_pattern, factory_pattern
3. **Developer answers** for multi-tenancy, IDOR safe/vulnerable patterns

**Key: use the EXACT patterns found in the project's existing test files**, not generic templates. If existing tests use `login(user, pessoa)`, the adapter must use exactly that. If tests use `create(:factory_name, user: user)`, the adapter must use exactly that.

**Framework mappings** — use these as fallbacks when existing test patterns are not found:

**Rails:**
- file_patterns: `app/models/`, `app/controllers/`, `app/views/`, etc.
- auth: `login(user, pessoa)` or `sign_in(user)` (from devise)
- idor: `grep 'params\[:.*_id\]'`, safe = `current_user.*.find()`, vulnerable = `Model.find()`
- test: `bundle exec rspec`, `_spec.rb`, `spec/`
- assertions: `have_received`, `be_present`, `have_http_status`, `allow().to receive`

**Django / DRF:**
- file_patterns: `*/models.py`, `*/views.py`, `*/serializers.py`, `*/templates/`
- auth: `self.client.force_authenticate(user)` or `self.client.login()`
- idor: `kwargs["pk"]` or `request.data.get("id")`, safe = `queryset.filter(owner=request.user)`, vulnerable = `Model.objects.get(pk=pk)`
- test: `pytest` or `python manage.py test`, `test_*.py`
- assertions: `mock.assert_called`, `assertIsNotNone`, `assertEqual`

**Express / Fastify / NestJS:**
- file_patterns: `src/routes/`, `src/controllers/`, `src/models/`, `pages/api/`
- auth: `request(app).set('Authorization', token)`
- idor: `req.params.id` without `where: { userId }`, vulnerable = `Model.findByPk(id)`
- test: `jest` or `vitest`, `.test.ts`, `__tests__/`
- assertions: `toHaveBeenCalled`, `toBeTruthy`, `toEqual`

**Next.js:**
- file_patterns: `app/` or `pages/`, `components/`, `lib/`, `pages/api/` or `app/api/`
- auth: depends on next-auth/clerk/lucia — extract from code
- idor: API route handlers with `params.id` without user scoping
- test: `jest` or `vitest`

**FastAPI:**
- file_patterns: `app/routers/`, `app/models/`, `app/schemas/`
- auth: `client.headers = {"Authorization": f"Bearer {token}"}`
- idor: path parameter `id` without `filter(owner_id=current_user.id)`
- test: `pytest`, `test_*.py`
- assertions: `mock.assert_called`, `assert response.json()["field"] == value`

**Go (Gin/Echo/Fiber/Chi):**
- file_patterns: `internal/handlers/`, `internal/models/`, `internal/services/`, `cmd/`
- auth: middleware mock or test JWT
- idor: `c.Param("id")` without user scoping
- test: `go test ./...`, `_test.go`
- assertions: `assert.Equal`, `assert.Nil`, `require.NoError` (testify)

**Phoenix (Elixir):**
- file_patterns: `lib/*/controllers/`, `lib/*/views/`, `lib/*/live/`, `lib/*/schemas/`
- auth: `conn |> log_in_user(user)` or similar plug
- idor: `Repo.get!(Model, id)` without user scoping, safe = `Repo.get_by!(Model, id: id, user_id: user.id)`
- test: `mix test`, `test/`, `_test.exs`
- assertions: `assert`, `refute`, `assert_redirected_to`

**Laravel (PHP):**
- file_patterns: `app/Http/Controllers/`, `app/Models/`, `app/Services/`, `routes/`
- auth: `$this->actingAs($user)` or `Sanctum::actingAs($user)`
- idor: `Model::find($id)` without user scoping, safe = `$user->models()->findOrFail($id)`
- test: `php artisan test` or `./vendor/bin/phpunit`, `tests/`, `Test.php`
- assertions: `assertStatus`, `assertJson`, `assertDatabaseHas`

**Spring Boot (Java/Kotlin):**
- file_patterns: `src/main/java/**/controller/`, `src/main/java/**/service/`, `src/main/java/**/repository/`
- auth: `@WithMockUser` or `SecurityMockMvcRequestPostProcessors`
- idor: `repository.findById(id)` without user scoping
- test: `mvn test` or `gradle test`, `src/test/`, `Test.java`
- assertions: `assertEquals`, `assertNotNull`, `assertThrows`, `mockMvc.perform()`

For frameworks not listed: use the detected patterns from reading files and ask the developer for specifics.

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

[MUST] Use the EXACT patterns extracted from existing test files — same auth helper, same factory calls, same assertion style.

#### 3.3 Generate `config.yml`

```yaml
# Generated by /qa-setup on YYYY-MM-DD
stack:
  language: <detected>
  language_version: <detected or null>
  framework: <detected>
  framework_version: <detected or null>
  orm: <detected or null>
  database: <detected or null>
  api_style: <rest|graphql|grpc>

paths:
  report_dir: qa-reports
  temp_dir: /tmp/qa-specs
  source_dirs: "<detected source directories>"

test:
  runner: <detected>
  command: "<detected test command>"
  dir: "<detected test directory>"
  file_pattern: "<detected pattern>"

auth:
  library: <detected or null>
  strategy: <session|token|oauth|none>
  login_helper: "<detected or from question>"
  tenant_field: "<from question>"
  other_tenant_helper: "<from question>"

tools:
  coverage: <detected or null>
  security: [<detected list>]
  linter: <detected or null>
  formatter: <detected or null>

ci:
  platform: <detected or null>
  target_branches: [main]

report:
  language: <from question>
```

#### 3.4 (Optional) Install CI

If CI platform was detected:
- Ask "Do you want QA gates added to your existing [platform] CI?"

If dev chose GitHub Actions:
1. Read `ci/qa-gate.yml` template
2. Replace placeholders with values from `config.yml`
3. Write to `.github/workflows/qa-gate.yml`
4. Same for `ci/qa-cleanup.yml`

If dev chose GitLab CI:
1. Generate equivalent `.gitlab-ci.yml` stage

If no CI detected and dev wants CI:
1. Ask which platform
2. Generate from template

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

Detected stack:
  [language] [version] + [framework] [version] + [test_runner]
  DB: [database] via [orm]
  Auth: [auth_lib] ([strategy])
  CI: [ci_platform]

Run your first QA:
  /qa-diff

Or specify a PR:
  /qa-diff 123
```

## Edge Cases

- **Monorepo**: If multiple `package.json` / `Gemfile` detected at different paths, ask which one is the primary app. Use that app's dependencies for detection.
- **Multiple languages**: If both `Gemfile` and `package.json` exist (e.g., Rails + frontend), detect the primary backend and note the frontend. Generate adapter for the backend; frontend testing is noted as optional in adapter.
- **No test runner detected**: Ask the developer what they use. If none, warn that qa-diff requires a test runner and suggest setting one up first.
- **Custom frameworks**: If no known framework detected, ask for file layout and test patterns. Generate a minimal adapter with what's known.
- **No existing tests**: If the test directory is empty or doesn't exist, fall back to framework-specific templates from the mappings above. Warn that patterns are generic and may need adjustment.
- **Auth not detectable**: If no auth library is found in deps AND no auth patterns in test files, ask the developer all auth questions (Q2 from question bank).
- **Multiple databases**: If docker-compose shows multiple databases (e.g., postgres + redis + mongo), detect the primary DB (the one the ORM connects to) and note others as auxiliary services.
