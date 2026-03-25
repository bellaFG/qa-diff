---
name: Onboarding Question Bank
description: Structured questions for /qa-setup onboarding — most are conditional, asked only when auto-detection can't resolve them.
version: 2.0.0
---

# Onboarding Question Bank

Questions organized by topic. Most questions are **conditional** — only asked when auto-detection (`install/detect-stack.md`) fails to resolve the information.

## Detection-First Approach

The onboarding process reads real files in the repo before asking any questions. This means:

- If `Gemfile` contains `rails` → we know the framework. No need to ask.
- If existing test files contain `login(user, pessoa)` → we know the auth helper. No need to ask.
- If `spec/factories/` exists with FactoryBot files → we know the factory pattern. No need to ask.

**Only ask what can't be inferred from the codebase.**

---

## Q1 — Stack Confirmation

**Always asked.** Presents auto-detection results for confirmation.

```
header: "Stack Detection"
question: |
  Detected Stack:
    Language:       [language] [version]
    Framework:      [framework] [version]
    ORM/DB:         [orm] + [database]
    Test Runner:    [test_runner] (dir: [test_dir])
    Coverage:       [coverage_tool or "not detected"]
    Security:       [security_tools or "none detected"]
    Linter:         [linter] + [formatter]
    Auth:           [auth_lib] ([strategy])
    Factories:      [factory_lib] (dir: [factory_dir])
    API:            [api_style]
    CI:             [ci_platform]

    Auth helper (from tests): [extracted or "not found"]
    Factory pattern (from tests): [extracted or "not found"]

  Is this correct?
options:
  - label: "Yes, correct"
    description: "Proceed with detected stack"
  - label: "No, let me adjust"
    description: "I'll correct specific items"
```

If "No": ask which specific items are wrong and accept corrections.

---

## Q2 — Authentication Helper

**Asked only if:** auth library detected BUT exact test helper NOT found in existing test files.

```
header: "Auth Helper"
question: "We detected [auth_lib] but couldn't find the test helper pattern in your test files. How do you authenticate in tests?"
options:
  - label: "sign_in(user)"
    description: "Devise-style helper"
    show_if: auth_lib == devise
  - label: "login(user)"
    description: "Custom login helper"
  - label: "client.force_authenticate(user=user)"
    description: "DRF-style authentication"
    show_if: framework == django
  - label: "Custom"
    description: "Let me type the exact helper"
```

**Asked only if:** NO auth library detected at all.

```
header: "Auth"
question: "How does authentication work in your tests?"
options:
  - label: "Session-based"
    description: "Login form or helper that sets session cookie"
  - label: "Token-based"
    description: "JWT, API key, or bearer token in headers"
  - label: "No auth"
    description: "Public API, no authentication required"
  - label: "Custom"
    description: "Let me describe my auth pattern"
```

Follow-up (if session or token):
```
header: "Auth helper"
question: "What's the test helper to authenticate a user? (e.g., login(user), sign_in(user), client.force_authenticate(user))"
options: [free text]
```

---

## Q3 — Multi-Tenancy

**Always asked.** Business logic — cannot be reliably detected from code alone.

```
header: "Tenancy"
question: "Does your app isolate data per user/organization (multi-tenancy)?"
options:
  - label: "FK-based (user_id, account_id)"
    description: "Records have a foreign key to the owning user/account"
  - label: "Org-based (organization_id)"
    description: "Records belong to an organization/workspace"
  - label: "Schema-based"
    description: "Separate database schema per tenant"
  - label: "No multi-tenancy"
    description: "Single-tenant app, all data is shared"
```

**Follow-up (if multi-tenant):**
```
header: "Tenant FK"
question: "What field scopes records to a tenant? (e.g., user_id, organization_id, account_id)"
options: [free text]
```

```
header: "Other tenant"
question: "How do you create a second tenant in tests? (e.g., create(:user), User.create(org: other_org))"
options: [free text]
```

---

## Q4 — Test Data Creation

**Asked only if:** NO factory library detected AND no factory/fixture patterns found in existing test files.

```
header: "Test data"
question: "How do you create test data?"
options:
  - label: "Factories"
    description: "FactoryBot, factory_boy, fishery, or similar"
  - label: "Fixtures"
    description: "YAML/JSON fixture files loaded before tests"
  - label: "Helper functions"
    description: "Custom builder/helper functions"
  - label: "Inline creation"
    description: "Direct DB inserts in test setup"
```

**Follow-up (if factories):**
```
header: "Factory lib"
question: "Which factory library? (e.g., FactoryBot, factory_boy, fishery)"
options: [free text]
```

---

## Q5 — Coverage Tool

**Asked only if:** NOT auto-detected.

```
header: "Coverage"
question: "Do you have a test coverage tool configured?"
options:
  - label: "Yes, let me specify"
    description: "I'll provide the tool name"
  - label: "No coverage tool"
    description: "Skip coverage analysis"
```

---

## Q6 — Security Tools

**Asked only if:** NOT auto-detected.

```
header: "Security"
question: "Do you have SAST (Static Application Security Testing) tools?"
options:
  - label: "Yes, let me specify"
    description: "I'll provide the tool names"
  - label: "No security tools"
    description: "Skip security scanning"
```

---

## Q7 — CI Setup

**If CI detected:**
```
header: "CI"
question: "Detected [ci_platform] CI. Do you want QA gates added to your existing CI?"
options:
  - label: "Yes, add QA gates"
    description: "Add qa-gate and qa-cleanup workflows"
  - label: "Skip CI"
    description: "I'll set up CI later"
```

**If NO CI detected:**
```
header: "CI"
question: "No CI detected. Do you want to set up CI gates for QA reports?"
options:
  - label: "GitHub Actions (recommended)"
    description: "Add qa-gate.yml and qa-cleanup.yml workflows"
  - label: "GitLab CI"
    description: "Add QA gate stage to .gitlab-ci.yml"
  - label: "Skip CI"
    description: "I'll set up CI later"
```

---

## Q8 — Report Language

**Always asked.** User preference.

```
header: "Language"
question: "What language should QA reports be written in?"
options:
  - label: "English"
    description: "Reports, findings, and triage messages in English"
  - label: "Portuguese (pt-BR)"
    description: "Reports, findings, and triage messages in Portuguese"
  - label: "Spanish"
    description: "Reports, findings, and triage messages in Spanish"
```

---

## Q9 — Monorepo Target

**Asked only if:** monorepo detected (multiple apps/packages).

```
header: "Monorepo"
question: "Monorepo detected with multiple apps/packages. Which is the primary target for QA?"
options:
  - label: "[app_1]"
    description: "Path: [path_1]"
  - label: "[app_2]"
    description: "Path: [path_2]"
  - label: "Other"
    description: "Let me specify the path"
```

---

## Ordering

1. Run full auto-detection (Step 1 of onboarding.md)
2. Present Q1 (stack confirmation with ALL detected info)
3. Ask only the conditional questions that apply:
   - Q2 (auth) — only if not fully detected
   - Q3 (tenancy) — always
   - Q4 (test data) — only if not detected
   - Q5 (coverage) — only if not detected
   - Q6 (security) — only if not detected
   - Q7 (CI) — adapted based on detection
   - Q8 (language) — always
   - Q9 (monorepo) — only if monorepo detected

Batch questions when possible using `AskUserQuestion` (max 4 per call) to minimize interruptions.

**Best case (everything detected):** only 3 questions — Q1 (confirm), Q3 (tenancy), Q8 (language).

**Worst case (nothing detected):** all 8 questions — Q1 through Q8.
