# Exam Prep CI Workflow

Copy the YAML below to `.github/workflows/exam-prep-ci.yml`. (It is kept here as Markdown because the
repo only allows `.md` files outside `.github/workflows`, and the Claude GitHub App can't write to
workflow files.)

It runs on every push/PR to `main` and checks:

1. **Markdown lint**: consistent formatting of study notes.
2. **Link check**: broken links to official docs.
3. **Study progress**: prints how many topics in `README.md` are checked off.

```yaml
name: Exam Prep CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  lint:
    name: Markdown lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: DavidAnson/markdownlint-cli2-action@v16
        with:
          globs: "**/*.md"
          config: |
            { "default": true, "MD013": false, "MD033": false, "MD041": false }

  links:
    name: Link check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: lycheeverse/lychee-action@v2
        with:
          args: --no-progress --accept 200,429 '**/*.md'
          fail: true

  progress:
    name: Study progress
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Count completed topics
        run: |
          done=$(grep -cE '^- \[[xX]\]' README.md || true)
          todo=$(grep -cE '^- \[ \]' README.md || true)
          total=$((done + todo))
          echo "Topics completed: $done / $total"
          echo "### Study progress: $done / $total topics" >> "$GITHUB_STEP_SUMMARY"
```
