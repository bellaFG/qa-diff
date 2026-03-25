---
name: QA Pipeline v1
description: Framework-agnostic adversarial QA pipeline — integration-only, IDOR audit, boundary map, H1-H10 sentinel, mutation testing, load testing.
version: 1.0.0
---

<!-- [MUST]=mandatory [SHOULD]=recommended [MAY]=optional [ADAPTER]=read from adapter/project.md -->

# QA Pipeline — Integration-Only, Framework-Agnostic

Autonomous QA pipeline for diffs. Max 30 min. Single agent.

## Principles

1. **QA only** — specs validate the PR and do NOT enter it. All in `${TMPDIR:-/tmp}/qa-specs/`
2. **Integration-only** — NEVER unit tests. Every spec is an integration test
3. **Real environment** — env vars, database, exports, background jobs, caches
4. **Adversarial** — try to break > prove it works
5. **Assertive** — 5 specs with exact value assertions > 20 specs with status-only checks
6. **IDOR and bugs** — unscoped parameter lookup = Bug F
7. **Zero internal mocks** — NEVER mock/stub internal models, controllers, auth. Mock ONLY external dependencies (SDKs, binaries, paid APIs)
8. **Sentinel H1-H10** — self-review before executing
9. **Time-boxing** — budget per PR_SIZE, max 3 rounds per file
10. **Model routing** — fast model for mechanical tasks, reasoning model for generation/analysis

## QA, Not Dev

The QA pipeline does NOT write or modify production code. It only tests and reports.

- ALL specs in `${TMPDIR:-/tmp}/qa-specs/`
- Findings (bugs, vulnerabilities, improvements, FRAGILE) go to the report
- In triage (Phase 7), present each finding via `AskUserQuestion` with 3 paths:
  - **Coder fixes** — pass finding context to the coder
  - **New card** — create an issue with finding context
  - **Ignore** — record decision in report

## Real Environment

[MUST] Full access to the development environment. NEVER remote environments.
[MUST] Database cleanup/transactions between specs.

## Model Routing

| Phase | Task | Model |
|-------|------|-------|
| 1 | Discovery | **fast** (haiku) |
| 2 | Planning | **reasoning** (sonnet) |
| 3 | Spec generation | **reasoning** (sonnet) |
| 3.4 | Sentinel H1-H10 | **reasoning** (sonnet) |
| 3.6 | Execution (test runner) | **fast** (haiku) |
| 3.7 | Coverage | **fast** (haiku) |
| 3.8 | Mutation testing | **reasoning** (sonnet) |
| 4 | Classify Findings | **reasoning** (sonnet) |
| 5 | Load testing (KR) | **reasoning** (sonnet) |
| 6 | Report | **fast** (haiku) |
| 7 | Triage (AskUserQuestion) | **reasoning** (sonnet) |

## Checkpoints

```
CHECKPOINT Phase 1: "Discovery: N files, PR_TYPE=X, SIZE=Y, Budget=Z, IDOR=[yes/no], KR=[yes/no]"
CHECKPOINT Phase 2: "Plan: N scenarios (N normal + N adversarial + N IDOR)"
CHECKPOINT Phase 3: "Specs: N created, N passing, N failing, coverage=X%"
CHECKPOINT Phase 3.8: "Mutation: score=X% (N killed / N total)"
CHECKPOINT Phase 4: "Findings: N bugs, N vulnerabilities, N improvements, N FRAGILE"
CHECKPOINT Phase 6: "Verdict: APPROVED/APPROVED_WITH_PENDING/BLOCKED"
CHECKPOINT Phase 7: "Triage: N coder, N card, N ignored"
```

---

## Phase 1 — Discovery

### 1.0 Detect incremental run

Before starting full discovery, check for a previous report:

```bash
BRANCH=$(git branch --show-current)
BRANCH_SAFE=$(echo "$BRANCH" | sed 's|/|-|g')
REPORT_DIR="[ADAPTER: paths.report_dir]"  # from config.yml
PREV_REPORT="${REPORT_DIR}/qa-report-${BRANCH_SAFE}.md"
```

If `PREV_REPORT` exists and contains `head_sha`:
1. Extract `PREV_SHA` from frontmatter
2. Calculate delta: `git diff --name-only ${PREV_SHA}..HEAD -- [ADAPTER: source_dirs]`
3. If delta only contains files already covered in previous report (dev fixed bugs without adding features):
   - `MODE=incremental`
   - Reuse discovery and plan from previous report
   - Execute ONLY specs for changed files + re-validate previous findings
   - Budget = half of original budget
4. If delta introduces new source files not present in previous report:
   - `MODE=full` — run full
5. If `PREV_REPORT` does not exist: `MODE=full`

[MUST] Report frontmatter must include `mode: full | incremental`.

[MUST] Incremental report INHERITS unaffected findings from previous (does not re-test what didn't change).

### 1.1 Determine the diff

[MUST] `git fetch origin` before any diff.

[MUST] Correct base:

```bash
if [ -n "$PR_NUMBER" ]; then
  BASE=$(gh pr view "$PR_NUMBER" --json baseRefName --jq '.baseRefName')
else
  BASE=$(gh pr view --json baseRefName --jq '.baseRefName' 2>/dev/null)
fi

if [ -z "$BASE" ]; then
  RELEASE=$(git branch -r --list 'origin/release/*' | tail -1 | xargs)
  if [ -n "$RELEASE" ]; then
    DIST_MAIN=$(git rev-list --count "$(git merge-base HEAD origin/main)..HEAD" 2>/dev/null || echo 9999)
    DIST_REL=$(git rev-list --count "$(git merge-base HEAD "$RELEASE")..HEAD" 2>/dev/null || echo 9999)
    [ "$DIST_REL" -lt "$DIST_MAIN" ] && BASE=$(echo "$RELEASE" | sed 's|origin/||')
  fi
  BASE=${BASE:-main}
fi
```

| Input | Command |
|-------|---------|
| PR number | `gh pr diff <N> --name-only` + `gh pr diff <N>` |
| Branch | `git diff origin/<BASE>...<branch> --name-only` |
| HEAD | `git diff origin/<BASE>...HEAD --name-only` |

### 1.2 Categorize and size

[ADAPTER: file_patterns] — read category-to-glob mappings from the project adapter.

Default categories (adapter overrides):

| Category | Description |
|----------|-------------|
| model | Data models, entities, domain objects |
| endpoint | Controllers, handlers, routes, resolvers |
| service | Services, calculators, generators, builders |
| view | Templates, components, pages |
| script | Client-side scripts, frontend logic |
| migration | Database migrations, schema changes |
| helper | Helpers, utilities |
| job | Background jobs, workers, tasks |
| config | Configuration files |
| test | Existing test files |

PR_SIZE by files in source directories:

| Files | Size | Budget |
|-------|------|--------|
| 1-2 | Micro | 10 min |
| 3-5 | Small | 20 min |
| 6-10 | Medium | 30 min |
| 11+ | Large | 30 min |

Max 3 rounds per file. Large: SECURITY > calculations > logic > CRUD.

### 1.3 Read context

For each changed file (except tests):
1. Read complete file
2. Read existing tests
3. Factory/fixture discovery: [ADAPTER: factory_discovery_command]

[SHOULD] If the project has bounded context docs → load relevant context.
[MAY] Read project learning notes if available.

### 1.4 Removed tests

Test removed without its production file being removed → flag for restoration.

### 1.6 PR_TYPE

Precedence: SECURITY > LOGIC > ENDPOINT > VIEW > CONFIG.

| Type | When |
|------|------|
| **SECURITY** | SQL interpolation, unsafe reflection, eval/exec with external data, unscoped param lookup |
| **LOGIC** | Models, jobs, services with calculations |
| **ENDPOINT** | Controllers/handlers CRUD, authorization |
| **VIEW** | Templates, components, client scripts |
| **CONFIG** | Migrations, config, locales |

If diff changed authorization middleware/decorators → `AUTHORIZATION_CHANGED = true`.

| Type | Tests | Coverage | SAST | Adversarial | Mutation |
|------|-------|----------|------|-------------|---------|
| SECURITY | yes | line | yes | FULL | no |
| LOGIC | yes | line | no | STANDARD | yes (if calculations) |
| ENDPOINT | yes | line | no | STANDARD | no |
| VIEW | yes | no | no | INPUT_ONLY | no |
| CONFIG | smoke | no | no | no | no |

### 1.7 IDOR Scan

[ADAPTER: idor_detection] — read scan command, safe patterns, and vulnerable patterns.

Generic concept:
- For each endpoint handler in the diff, search for parameter-based record lookup
- **SAFE**: lookup scoped to current user/tenant/org (e.g., `current_user.items.find(id)`)
- **VULNERABLE**: global/unscoped lookup (e.g., `Item.find(id)`, `db.query("SELECT * FROM items WHERE id = ?", id)`)

IDOR found → `IDOR_DETECTED = true`, spec mandatory (Phase 3.5.5).

### 1.8 Detect performance branch

Detect by **branch name**, **PR title/body**, and **diff context**:

```bash
BRANCH=$(git branch --show-current)
LOAD_TEST_CANDIDATE=false

# 1. Branch name
if echo "$BRANCH" | grep -qiE '(^kr[/-]|/kr[/-]|_kr[/-]|perf|performance|optimize)'; then
  LOAD_TEST_CANDIDATE=true
fi

# 2. PR title/body
if [ -n "$PR_NUMBER" ]; then
  PR_TITLE=$(gh pr view "$PR_NUMBER" --json title --jq '.title')
  PR_BODY=$(gh pr view "$PR_NUMBER" --json body --jq '.body')
  if echo "$PR_TITLE $PR_BODY" | grep -qiE '\bKR\b|performance|optimiz|benchmark|slow|N\+1|cache'; then
    LOAD_TEST_CANDIDATE=true
  fi
fi
```

3. **Diff context** — evaluate if changes suggest performance impact:
   - Eager loading added or removed
   - Batch processing added
   - Counter caches, bulk operations
   - Queries rewritten, indexes added
   - Caching added or modified
   - N+1 removal, loop optimization

If any criterion is true → `LOAD_TEST_CANDIDATE=true` → offer load testing in Phase 5 via `AskUserQuestion`.

---

## Phase 2 — Planning

### 2.1 Map scenarios

**Only test what changed.**

| Priority | Type |
|----------|------|
| 1 [MUST] | Regression |
| 2 [MUST] | IDOR (cross-tenant) |
| 3 [MUST if SECURITY] | Attack payloads |
| 4 [MUST] | Adversarial (nil, wrong type, empty, huge) |
| 5 [SHOULD] | Error cases |
| 6 [SHOULD] | Edge/boundary |
| 7 [MUST if calculations] | Invariants |
| 8 [SHOULD if calculations] | Mutation testing |

Cap: max 10 scenarios per file.

### 2.1.1 Boundary Value Map

For each new/changed parameter:

```
| Field | Type | nil | 0/empty | invalid | max_limit | other tenant |
```

Legend: N = normal, A = adversarial, I = IDOR. Each marked cell → `context` in Phase 3. Max 10 fields.

| Type | nil | 0/empty | invalid | max_limit |
|------|-----|---------|---------|-----------|
| Integer/Decimal | `nil`/`null` | `0`, `-1` | `"abc"` | `999_999_999`, `0.01` |
| String | `nil`/`null` | `""` | `"<script>alert(1)</script>"` | 10,000 chars |
| Date | `nil`/`null` | `""` | `"not-a-date"` | `"1970-01-01"`, `"2099-12-31"` |
| FK (ID) | `nil`/`null` | `0` | `999999` | ID from another tenant (I) |
| Boolean | `nil`/`null` | `""` | `"true"` (string) | — |
| Enum/Status | `nil`/`null` | `""` | `"INVALID"` | value outside enum |

### 2.2 Spec type

ALL are integration. Never unit.

[ADAPTER: spec_type_mapping] — read from adapter which spec type to use per source category:

| Source changed | Spec type (generic) |
|---------------|---------------------|
| model/entity | Request/API spec (via HTTP). Model spec only if no endpoint exists |
| endpoint/handler | Endpoint/controller spec |
| helper/utility | Integration spec (within endpoint context) |
| job/worker | Job spec with real execution |
| view/component | Endpoint spec (verify response body) |

### 2.3 Present plan (SINGLE INTERRUPTION)

```
QA Plan — [branch]
TYPE: [X] · SIZE: [Y] · Budget: [Z min] · IDOR: [yes/no] · KR: [yes/no] · Mutation: [yes/no]

Boundary Map:
| Field | Type | nil | 0/empty | invalid | other tenant |
| supplier_id | FK | N | N | A | I |

IDOR Scan:
| param | Pattern | Status |
| :supplier_id | unscoped lookup | IDOR |

Specs (ALL in /tmp/):
[file.ext]
  N [scenario] · endpoint spec
  A [scenario] · endpoint spec
  I [scenario IDOR] · endpoint spec

[s] execute · [n] adjust
```

If dev reduces plan: list risks of removed scenarios before accepting.

---

## Phase 3 — Execution (zero interruptions until report)

### 3.1 Calibrate style

Read 2 existing tests of the same type. Replicate style, conventions, auth, fixtures/factories.

### 3.2 Write specs

ALL in `${TMPDIR:-/tmp}/qa-specs/`. Execute with [ADAPTER: test_execution.command].

```
/tmp/qa-specs/
  endpoints/    (or controllers/, requests/)
  models/       (only complex calculations without endpoint)
  load/         (only performance branches)
```

[ADAPTER: auth_pattern] — read auth setup and cross-tenant patterns.
[ADAPTER: factory_pattern] — read how to create test data with real persistence.

Rules:
- ALWAYS use real persistence (create records in DB, never in-memory stubs)
- NEVER mock authentication — use real auth helpers
- NEVER use in-memory doubles for internal models — use real records
- Mock ONLY external dependencies without global stubs (SDKs, binaries, paid APIs)
- Follow AAA pattern (Arrange — Act — Assert)

### 3.3 Classify by level

| Level | How to test |
|-------|-------------|
| **CRITICAL** (SQL, calculations, security, IDOR) | Exact value assertions, boundary, cross-tenant |
| **IMPORTANT** (filters, scopes, conditionals) | Filtered/modified result verification |
| **COVERAGE** (render, redirect) | Basic status check |

Status-only assertions are acceptable ONLY for COVERAGE level.

### 3.4 Sentinel H1-H10

Scan EVERY test case. Failed → fix BEFORE running.

[REF] Load `core/SENTINEL.md` for full check definitions and adapter-provided detection patterns.

### 3.5 Adversarial — TRY TO BREAK

[MUST for SECURITY/LOGIC/ENDPOINT] [MAY for VIEW]

[REF] Load `core/ADVERSARIAL.md` for full methodology.

#### 3.5.1 Payloads

| Type | [MUST] | [SHOULD] |
|------|--------|----------|
| String | `nil`, `""`, `"<script>alert(1)</script>"` | 10,000 chars, emoji |
| Integer/Decimal | `nil`, `""`, `"abc"`, `-1`, `0` | `999_999_999` |
| Date | `nil`, `""`, `"not-a-date"` | `"31/02/2026"` |
| ID (FK) | `nil`, `0`, `999999` | — |
| Boolean | `nil`, `""`, `"true"` (string) | — |

#### 3.5.2 Three mandatory assertions

For every adversarial test:

```pseudocode
1. action(invalid_input) does NOT raise unhandled exception
2. response status is in [200, 302, 400, 422] (friendly)
3. no corrupted data persisted in database
```

#### 3.5.3 Invariants

[MUST for calculations] Validate invariant (e.g., sum of parts == total). Max 2 per file.

#### 3.5.4 State corruption

[SHOULD for update/destroy] Record deleted between operations.

#### 3.5.5 IDOR

[MUST for each parameter-based lookup in endpoint]

[ADAPTER: idor_template] — read cross-tenant spec template.

Generic pattern:
```pseudocode
context "IDOR: resource from another tenant"
  create resource belonging to other_tenant
  attempt to access/modify via current_tenant's session
  assert: access denied (redirect or 403/404)
  assert: resource not modified
```

#### 3.5.6 Self-Audit

Before executing, re-read EVERY spec against:
1. Boundary Map: marked cell has corresponding test context?
2. IDOR: parameter-based lookup has spec with cross-tenant?
3. H1-H10: full scan
4. Zero internal mocks? (only external deps allowed)
5. Zero in-memory stubs?

`SELF-AUDIT: N checks, N fixed, N OK`

### 3.6 Execute

[ADAPTER: test_execution.command] — run all specs in temp directory.

[ADAPTER: test_execution.coverage_flag] — enable coverage collection.

### 3.7 Coverage

Target: 95% of diff lines via [ADAPTER: coverage.result_path].

[ADAPTER: coverage.extraction_script] — script to extract coverage for diff lines.

### 3.8 Mutation Testing

[MUST for LOGIC with calculations] [MAY for others]

[ADAPTER: mutation_testing.command] — mutation testing tool command.

If no mutation testing tool available → property-based testing (loop with random inputs).

Score: >= 80% GOOD, 60-79% MEDIUM, < 60% WEAK.

### 3.9 Classify failures

| Type | Action |
|------|--------|
| A — Incorrect spec | Fix spec (QA fixes its own spec) |
| B — Real bug | Register as finding → Phase 7 |
| C — Missing setup | Fix setup (QA fixes its own spec) |
| D — Environment | Blocker |
| E — Input causes exception | Register as finding → Phase 7 |
| F — IDOR / Vulnerability | Register as finding → Phase 7 |

### 3.10 Coverage cycle

Max 2 rounds. Identify branch/rescue/guard → add test context → re-execute.

### 3.11 Branch analysis

For each new `if`/`else`/`switch`/`case`/`catch`/`rescue`: spec covering both branches.

### 3.12 Security tools

[ADAPTER: security_tools] — run SAST and security linters.

If no tools configured, skip.

---

## Phase 4 — Classify Findings

QA does NOT fix code. It classifies and presents to the dev.

### 4.1 Finding types

| Type | Description | Scope |
|------|-------------|-------|
| **Bug** | PR code with wrong behavior | PR |
| **Vulnerability** | IDOR, SQL injection, XSS, etc | PR |
| **Improvement** | Code works but could be better | PR or tech debt |
| **FRAGILE** | Works today but will break | Adjacent code |

### 4.2 FRAGILE

[REF] See `core/FINDINGS.md` for subtypes and severity.

---

## Phase 5 — Load Testing

[MUST] If `LOAD_TEST_CANDIDATE=true`: ask via `AskUserQuestion`. NOT part of the automatic pipeline.

```
Performance context detected ([reason: branch name / PR mentions performance / diff changes queries/cache/batch]).

Run load test?

Endpoints/methods changed:
- [list]

[s] execute · [n] skip
```

[ADAPTER: load_testing] — read load testing templates and tools.

| Metric | GOOD | ACCEPTABLE | BAD |
|--------|------|------------|-----|
| Avg | < 100ms | 100-500ms | > 500ms |
| P95 | < 200ms | 200-1000ms | > 1s |
| N+1 | No | — | Yes |

---

## Phase 6 — Report

```
## QA REPORT — [APPROVED/APPROVED_WITH_PENDING/BLOCKED]

[branch] · [date] · TYPE: [X] · SIZE: [Y] · Budget: [used]/[total] min · Mode: [full/incremental]

### Summary
Specs: N total · N failures · ALL in /tmp/
Coverage: N% · IDOR: N · Bugs: N · FRAGILE: N
H1-H10: [checkmarks]
Mutation: score=N% (if applicable)

### IDOR Audit
| param | Pattern | Status |

### Coverage
| File | % diff | Lines |

### Adversarial
| Payload | Result |

### Mutation Testing (if applicable)
| File | Mutants | Killed | Alive | Score |

### Load Testing (if KR)
| Metric | Value | Status |

### Findings (if any)
**Finding N** · [file:line] · [Bug/Vulnerability/Improvement/FRAGILE] · [CODER/CARD/IGNORED]

### Specs created (temporary)
- [/tmp/qa-specs/path] (N examples, N failures)

### Next steps
[project-specific next commands]
```

Verdict:
- **APPROVED** — zero open findings
- **APPROVED_WITH_PENDING** — findings exist but dev ignored/deferred in triage. QA obligation fulfilled
- **BLOCKED** — environment issues, >50% of files blocked

---

## Phase 7 — Triage

[MUST] If findings exist: a single `AskUserQuestion` with ALL findings grouped.

For each finding, present:
- Title, file:line, description (actual vs expected)
- Type (Bug / Vulnerability / Improvement / FRAGILE)
- Scope: belongs to the PR or outside the PR?

3 paths per finding:
1. **[C] Coder fixes** — pass complete finding context to the coder
2. **[I] New card** — create issue with finding context
3. **[X] Ignore** — record decision in report

Format:

```
QA Findings:

**Bug 1** · [file:line] · Vulnerability
IDOR: `Model.find(params[:id])` without tenant scope
Expected: scoped lookup via current user

**Bug 2** · [file:line] · Bug
`value` accepts string without validation, causes TypeError
Expected: validation or type coercion with fallback

**Improvement 1** · [file:line] · FRAGILE-CRITICAL
Broad exception catch without message — user gets empty error

For each item: [C] coder fixes · [I] new card · [X] ignore
Ex: "1C 2C 3I" = bug 1 and 2 for coder, improvement 1 becomes card
```

[MUST] After dev response, execute chosen actions:
- [C] → pass finding context to coder
- [I] → create issue with finding context
- [X] → record in report as ignored

---

## Phase 8 — Persist

### 8.1 Artifact

Sanitize branch name: `BRANCH_SAFE=$(echo "$BRANCH" | sed 's|/|-|g')`.

Save to `[ADAPTER: paths.report_dir]/qa-report-${BRANCH_SAFE}.md` with frontmatter:

```yaml
---
artifact: qa-diff-report
branch: <branch name>
date: YYYY-MM-DD
head_sha: <git rev-parse HEAD>
verdict: APPROVED | APPROVED_WITH_PENDING | BLOCKED
pr_type: SECURITY | LOGIC | ENDPOINT | VIEW | CONFIG
pr_size: Micro | Small | Medium | Large
idor_detected: true | false
mode: full | incremental
---
```

[MUST] `head_sha` must be the SHA of HEAD at execution time. CI uses this field to validate that code hasn't changed after QA.

### 8.2 Feedback loop

Ignored finding → record as false positive in project learnings.

### 8.3 Cleanup

`rm -rf "${TMPDIR:-/tmp}/qa-specs"`
