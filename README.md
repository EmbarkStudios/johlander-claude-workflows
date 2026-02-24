# johlander-claude-workflows

Reusable GitHub Actions workflows for CI/CD and Claude Code integration.

## Available Workflows

### Claude Code Review (`claude-code-review.yml`)

Automatic AI-powered code review on pull requests using the Claude Code Review plugin.

**Usage:**

```yaml
# .github/workflows/claude-code-review.yml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]

jobs:
  review:
    uses: EmbarkStudios/johlander-claude-workflows/.github/workflows/claude-code-review.yml@main
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

**Inputs:**

| Input | Default | Description |
|---|---|---|
| `max_turns` | `30` | Maximum number of Claude turns |
| `review_prompt` | _(auto-generated)_ | Custom review prompt |
| `allowed_tools` | `Bash(gh:*),Bash(grep:*),Bash(sed:*),Read,Glob,Grep` | Claude allowed tools |

### Claude Code Interactive (`claude-interactive.yml`)

Respond to `@claude` mentions in issues, PR comments, and reviews.

**Usage:**

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
    uses: EmbarkStudios/johlander-claude-workflows/.github/workflows/claude-interactive.yml@main
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

**Inputs:**

| Input | Default | Description |
|---|---|---|
| `claude_args` | _(empty)_ | Additional Claude CLI arguments |

### Node CI (`node-ci.yml`)

CI pipeline for Node.js/TypeScript projects: linting, type-checking, testing, security audit, and build validation.

**Usage:**

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [master]
  push:
    branches: [master]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    uses: EmbarkStudios/johlander-claude-workflows/.github/workflows/node-ci.yml@main
    with:
      node_version: '20'
      lint_command: 'npm run lint'
      typecheck_command: 'npx tsc --noEmit'
      # build_command: 'npm run build'
      # test_command: 'npm test'
```

**Inputs:**

| Input | Default | Description |
|---|---|---|
| `node_version` | `20` | Node.js version |
| `lint_command` | `npm run lint` | Lint command (empty to skip) |
| `typecheck_command` | `npx tsc --noEmit` | Type-check command (empty to skip) |
| `build_command` | _(empty)_ | Build command (empty to skip) |
| `test_command` | _(empty)_ | Test command (empty to skip) |
| `test_runs_on` | `ubuntu-latest` | Runner for test job |
| `security_audit` | `true` | Run npm audit |

## Setup

### Prerequisites

Each consuming repo needs the `CLAUDE_CODE_OAUTH_TOKEN` secret configured for Claude workflows.

### Adding to a repository

1. Create thin wrapper workflow files in your repo's `.github/workflows/` directory
2. Point them at the shared workflows using `uses: EmbarkStudios/johlander-claude-workflows/.github/workflows/<workflow>.yml@main`
3. Pass required secrets and customize inputs as needed

The trigger events (`on:`) must be defined in the calling workflow since reusable workflows only support `workflow_call` triggers.
