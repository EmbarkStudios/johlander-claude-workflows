# Repository Setup Guide

Step-by-step instructions for adding CI/CD and Claude Code workflows to a new repository.

## Prerequisites

- Repository must be in the `EmbarkStudios` GitHub organization
- You need push access to the repository

## Step 1: Add the CLAUDE_CODE_OAUTH_TOKEN secret

The Claude workflows require a `CLAUDE_CODE_OAUTH_TOKEN` repository secret.

**Option A — Copy from an existing repo (quickest):**

Get the token value from a team member who has it, then set it:

```bash
gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo EmbarkStudios/<repo-name>
```

Paste the token value when prompted.

**Option B — Install the Claude GitHub App:**

> **Note:** This also scaffolds default workflow files that you may need to delete afterward if you're using custom workflows.

Run the Claude Code `/install-github-app` slash command, then delete any scaffolded workflow files that conflict with your setup:

```bash
# Remove scaffolded files if they were created
gh api --method DELETE "repos/EmbarkStudios/<repo-name>/contents/.github/workflows/claude-code-review.yml" \
  -f message="Remove scaffolded workflow" \
  -f sha="$(gh api repos/EmbarkStudios/<repo-name>/contents/.github/workflows/claude-code-review.yml --jq .sha)"

gh api --method DELETE "repos/EmbarkStudios/<repo-name>/contents/.github/workflows/claude.yml" \
  -f message="Remove scaffolded workflow" \
  -f sha="$(gh api repos/EmbarkStudios/<repo-name>/contents/.github/workflows/claude.yml --jq .sha)"
```

## Step 2: Add workflow files

Create the following files in `.github/workflows/`. Pick the workflows you need.

### CI Pipeline (shared workflow)

This uses the shared `node-ci.yml` reusable workflow. Adjust inputs for your project.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]  # or master
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    uses: EmbarkStudios/johlander-claude-workflows/.github/workflows/node-ci.yml@main
    with:
      node_version: '22'        # match your project's Node version
      lint_command: 'npm run lint'  # empty string to skip
      typecheck_command: ''         # empty string to skip
      build_command: ''             # empty string to skip
      test_command: 'npm test'      # empty string to skip
      security_audit: true
```

### Claude PR Review (inline)

> **Why inline?** The EmbarkStudios org Actions policy blocks third-party actions (`anthropics/claude-code-action`) when called through reusable workflows. This must be defined inline.

```yaml
# .github/workflows/claude-review.yml
name: Claude PR Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]

jobs:
  review:
    if: github.event.pull_request.draft == false
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      issues: write
      id-token: write
      actions: read

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Run Claude Code Review
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          prompt: |
            Review this PR. Focus on:
            - Code quality and potential bugs
            - Security issues
            - Test coverage gaps
            Approve the PR if everything looks good, or request changes if there are issues.
```

### Claude Interactive (inline)

> **Why inline?** Same reason as above — third-party action restriction.

```yaml
# .github/workflows/claude.yml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude')) ||
      (github.event_name == 'issues' && (contains(github.event.issue.body, '@claude') || contains(github.event.issue.title, '@claude')))
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
      actions: read

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Run Claude Code
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          claude_args: "--model sonnet"
          additional_permissions: |
            actions: read
            contents: write
```

## Step 3: Verify

1. Open a PR on the repository
2. Check that CI runs (if configured)
3. Check that Claude PR Review triggers and posts a review
4. Comment `@claude` on the PR to verify interactive mode works

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `startup_failure` with 0 jobs | Missing `CLAUDE_CODE_OAUTH_TOKEN` secret | Add the secret (Step 1) |
| `startup_failure` on shared Claude workflows | Org policy blocks third-party actions in reusable workflows | Use inline workflow definitions instead |
| CI runs but Claude review doesn't | Secret exists but may be empty/invalid | Re-set the secret with a valid token |
| Duplicate review workflows running | `/install-github-app` created extra files | Delete the scaffolded `claude-code-review.yml` |
