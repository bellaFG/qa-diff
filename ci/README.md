# CI Gates for QA Diff

## GitHub Actions

Two workflow templates are provided:

### `qa-gate.yml` — Validates QA Report on PRs

Runs on every PR to your target branches. Checks:
1. QA report exists for the PR branch
2. Report branch matches the PR branch
3. `head_sha` is not stale (code hasn't changed after QA)
4. Verdict is valid (APPROVED, APPROVED_WITH_PENDING, BLOCKED)
5. Posts the report as a PR comment (creates or updates)

### `qa-cleanup.yml` — Cleans Up After Merge

Runs when a PR is merged. Deletes the QA report file to keep the repo clean.

## Setup

### Automatic (via /qa-setup)

If you chose GitHub Actions during `/qa-setup`, the workflows are already in `.github/workflows/`.

### Manual

1. Copy `qa-gate.yml` to `.github/workflows/qa-gate.yml`
2. Copy `qa-cleanup.yml` to `.github/workflows/qa-cleanup.yml`
3. Update the `REPORT_DIR` variable in both files to match your `config.yml` `paths.report_dir`
4. Update `branches` arrays to match your target branches
5. Update the `ref` in qa-cleanup.yml to match your default branch

## Configuration Points

| Setting | Location in YAML | Default | Must Match |
|---------|-----------------|---------|------------|
| Report directory | `REPORT_DIR` | `qa-reports` | `config.yml` → `paths.report_dir` |
| Target branches | `on.pull_request.branches` | `[main]` | Your merge targets |
| Default branch | `ref` in cleanup checkout | `main` | Your default branch |

## GitLab CI

No template provided yet. The equivalent stages would be:

```yaml
qa-gate:
  stage: validate
  rules:
    - if: $CI_MERGE_REQUEST_IID
  script:
    - BRANCH_SAFE=$(echo "$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME" | sed 's|/|-|g')
    - REPORT="qa-reports/qa-report-${BRANCH_SAFE}.md"
    - test -f "$REPORT" || (echo "QA not executed. Run /qa-diff first." && exit 1)
    # Add frontmatter validation similar to GitHub Actions version
```
