# GitHub Actions and CI Triggers for Claude Code (or Any Agent)

This guide explains how GitHub Actions works, how triggers decide when a workflow runs, and how to
wire up Claude Code or any other agent. Examples come from the workflows in this repo.

## 1. Core concepts

| Concept | What it is |
| --- | --- |
| **Workflow** | A YAML file in `.github/workflows/` that defines automation. |
| **Event / trigger** | Something that happens in GitHub (push, PR, comment, schedule) and starts a workflow. |
| **Job** | A group of steps that runs on one runner. Jobs run in parallel unless linked with `needs:`. |
| **Step** | A single shell command (`run:`) or a reusable action (`uses:`). |
| **Runner** | The VM that executes a job, such as `ubuntu-latest`. |
| **Action** | A reusable step, such as `actions/checkout@v4` or `anthropics/claude-code-action@v1`. |
| **Secret** | An encrypted value, read as `${{ secrets.NAME }}`. |

Flow: **event happens → matching workflows start → jobs run on runners → steps execute → results show
on the commit, PR, or issue.**

## 2. Triggers (`on:`)

### 2.1 Common triggers

```yaml
on:
  push:
    branches: [main]          # commits pushed to main
  pull_request:
    branches: [main]          # PRs targeting main
    types: [opened, synchronize, reopened]
  issues:
    types: [opened, assigned]
  issue_comment:
    types: [created]          # comments on issues AND PRs
  pull_request_review:
    types: [submitted]
  pull_request_review_comment:
    types: [created]          # inline comments on a PR diff
  schedule:
    - cron: "0 6 * * 1"       # Mondays 06:00 UTC
  workflow_dispatch:          # manual "Run workflow" button
```

### 2.2 Activity types

Each event has `types:` that narrow when it fires. For example, `pull_request` defaults to
`opened`, `synchronize` (new commits pushed) and `reopened`. Add `ready_for_review` if you want draft
PRs to be reviewed once they are marked ready.

### 2.3 Filters

- `branches` / `branches-ignore`: limit by branch name.
- `paths` / `paths-ignore`: run only when matching files change (useful to skip the agent on docs-only
  changes, or run it only on `src/**`).
- `if:` on a job or step: an expression that must be true. Filters on `on:` decide whether the workflow
  starts, and `if:` decides whether a job runs.

### 2.4 Trigger gotchas

- `issue_comment` fires for comments on both issues and PRs. Check
  `github.event.issue.pull_request` to tell them apart.
- Workflows for `issue_comment`, `issues`, and `pull_request_review*` run from the **default branch's**
  workflow file, not the PR branch.
- `pull_request` runs from forks get a read-only token and **no secrets**. This is intentional, so
  untrusted code can't read your API key. Avoid `pull_request_target` with a checkout of PR code unless
  you fully understand the risk.
- Events created by `GITHUB_TOKEN` do not trigger other workflows (this prevents loops). Use a GitHub
  App token or PAT if one workflow must trigger another.
- Changes to files under `.github/workflows/` can't be pushed by the Claude GitHub App. A human has to
  commit them.

## 3. Two patterns for running an agent

### 3.1 Interactive: respond to `@claude` mentions

Used by `.github/workflows/claude.yml` in this repo. A human mentions `@claude` in an issue, comment, or
review and the agent does what was asked, such as answering, editing code, or opening a PR.

```yaml
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
    # Only run when the trigger phrase is present
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude')) ||
      (github.event_name == 'issues' && contains(github.event.issue.body, '@claude'))
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

The `if:` guard matters. Without it the job would start on every comment and waste runner minutes.

### 3.2 Automatic: run on every PR

Used by `.github/workflows/claude-code-review.yml`. No human prompt is needed. The workflow fires on
PR events and passes a fixed `prompt:` to the agent.

```yaml
on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          prompt: "Review this PR for bugs and style issues."
```

### 3.3 Other triggers worth using with an agent

- **`schedule`**: nightly dependency checks, weekly docs freshness audits, stale-issue triage.
- **`workflow_dispatch`**: manual runs with `inputs:` for ad-hoc tasks.
- **`issues: [labeled]`**: start the agent when a label such as `ai-fix` is applied.
- **`workflow_run`**: run an agent after another workflow finishes, for example to diagnose a failed CI
  run. Pair it with `actions: read` permission so the agent can read the logs.

## 4. Using any other agent

Nothing in Actions is specific to Claude. For any agent or CLI:

1. Pick the trigger and an `if:` guard.
2. Check out the code.
3. Provide credentials through a secret, never hard-coded.
4. Run the agent through its action or a `run:` step.
5. Give it only the permissions it needs.

```yaml
steps:
  - uses: actions/checkout@v4
  - name: Run my agent
    env:
      AGENT_API_KEY: ${{ secrets.AGENT_API_KEY }}
      GH_TOKEN: ${{ github.token }}
    run: ./scripts/run-agent.sh "${{ github.event.issue.number }}"
```

Be careful when interpolating `${{ ... }}` values such as issue titles or comment bodies into `run:`
scripts. They are attacker-controlled text. Pass them through `env:` and quote them instead.

## 5. Permissions and security checklist

- Set `permissions:` explicitly at workflow or job level and start from `contents: read`.
- Grant write scopes (`contents: write`, `pull-requests: write`, `issues: write`) only to jobs that
  need them.
- `id-token: write` is needed by the Claude action for its OAuth/OIDC exchange.
- Store keys in repository or organization **Secrets**. Never print them in logs.
- Restrict who can trigger the agent. Only users with write access can trigger `claude-code-action`
  by default.
- Treat issue and PR text as untrusted input (prompt injection). Limit the agent's tools with
  `claude_args: '--allowed-tools ...'`.
- Pin third-party actions to a version tag or commit SHA.
- Use `concurrency:` to cancel superseded runs and cap cost:

```yaml
concurrency:
  group: claude-${{ github.event.pull_request.number || github.event.issue.number }}
  cancel-in-progress: true
```

## 6. Debugging triggers

| Symptom | Likely cause |
| --- | --- |
| Workflow never starts | File isn't on the default branch (for comment/issue events), wrong `branches`/`paths` filter, or a YAML error. |
| Starts but job is skipped | The `if:` expression was false, for example the trigger phrase is missing. |
| No secret available | The run came from a fork PR, or the secret name is misspelled. |
| Agent can't push or comment | Missing `permissions:` scope. |
| Agent can't edit workflows | GitHub App tokens can't modify `.github/workflows/`. |
| One workflow doesn't chain into another | Events from `GITHUB_TOKEN` don't trigger workflows. |

Tips: use the **Actions** tab to inspect each run, re-run with debug logging by setting the
`ACTIONS_STEP_DEBUG` secret to `true`, and print the event context with
`echo '${{ toJSON(github.event) }}'` while developing, then remove it.

## 7. This repo's workflows at a glance

| File | Trigger | Purpose |
| --- | --- | --- |
| `.github/workflows/claude.yml` | `issue_comment`, `issues`, `pull_request_review`, `pull_request_review_comment` with an `@claude` guard | Interactive agent. |
| `.github/workflows/claude-code-review.yml` | `pull_request` (opened, synchronize, ready_for_review, reopened) | Automatic PR review. |
| `.github/workflows/markdown-only.yml` | `pull_request` and `push` to `main` | Enforces that only Markdown files are tracked. |
| `docs/exam-prep-ci.md` | n/a (template) | Optional lint, link check and progress workflow to copy in. |

## 8. Exam takeaways

- Know the difference between event-driven (`pull_request`, `issue_comment`), scheduled, and manual
  triggers, and when each suits an agent.
- Know that `if:` guards, `paths` filters, and `concurrency` control cost and noise.
- Know least-privilege `permissions:`, secrets handling, and why fork PRs have no secrets.
- Know that agent input from issues and PRs is untrusted, so restrict the agent's tools.
