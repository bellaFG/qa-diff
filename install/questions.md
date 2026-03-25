---
name: Onboarding Question Bank
description: Structured questions for /qa-setup onboarding — stack, auth, tenancy, testing, CI.
version: 1.0.0
---

# Onboarding Question Bank

Questions organized by topic. The onboarding skill (`install/onboarding.md`) selects which questions to ask based on auto-detection results.

## Q1 — Stack Confirmation

**Always asked.**

```
header: "Stack"
question: "Detected [language] + [framework] + [test_runner]. Is this correct?"
options:
  - label: "Yes, correct"
    description: "Proceed with detected stack"
  - label: "No, let me specify"
    description: "I'll provide the correct stack details"
```

## Q2 — Authentication Model

**Always asked.**

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

**Follow-up (if session or token):**
```
header: "Auth helper"
question: "What's the test helper to authenticate a user? (e.g., login(user), sign_in(user), client.force_authenticate(user))"
options: [free text]
```

## Q3 — Multi-Tenancy

**Always asked.**

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

## Q4 — Test Data Creation

**Always asked.**

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

## Q5 — Coverage Tool

**Asked only if not auto-detected.**

```
header: "Coverage"
question: "Do you have a test coverage tool configured?"
options:
  - label: "Yes (auto-detected: [tool])"
    description: "Use the detected tool"
  - label: "Yes, different tool"
    description: "I use a different coverage tool"
  - label: "No coverage tool"
    description: "Skip coverage analysis"
```

## Q6 — Security Tools

**Asked only if not auto-detected.**

```
header: "Security"
question: "Do you have SAST (Static Application Security Testing) tools?"
options:
  - label: "Yes (auto-detected: [tools])"
    description: "Use detected tools"
  - label: "Yes, different tools"
    description: "I'll specify my security tools"
  - label: "No security tools"
    description: "Skip security scanning"
```

## Q7 — CI Platform

**Always asked.**

```
header: "CI"
question: "Do you want to set up CI gates for QA reports?"
options:
  - label: "GitHub Actions (recommended)"
    description: "Add qa-gate.yml and qa-cleanup.yml workflows"
  - label: "GitLab CI"
    description: "Add QA gate stage to .gitlab-ci.yml"
  - label: "Skip CI"
    description: "I'll set up CI later"
```

## Q8 — Report Language

**Always asked.**

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

## Ordering

Ask questions in this order: Q1 → Q2 → Q3 → Q4 → Q5 (if needed) → Q6 (if needed) → Q7 → Q8.

Batch questions when possible using `AskUserQuestion` with multiple questions (max 4 per call) to minimize interruptions.
