# DiffPal Review GitHub Action

Run DiffPal pull request review from GitHub Actions. The action installs the DiffPal CLI from npm by default, then runs `diffpal review github` with the pull request base and head revisions.

## Quick Start

```yaml
name: diffpal

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

concurrency:
  group: diffpal-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  review:
    name: review
    if: ${{ !github.event.pull_request.draft && github.event.pull_request.head.repo.full_name == github.repository }}
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      checks: write
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v6
        with:
          node-version: 22

      - name: Install Codex provider
        run: npm install --global @openai/codex@0.139.0 @normahq/codex-acp-bridge@1.6.3

      - name: Authenticate Codex
        run: printf '%s' "$OPENAI_API_KEY" | codex login --with-api-key
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}

      - name: Review pull request
        uses: diffpal/action@v1
        with:
          profile: ci
          base: ${{ github.event.pull_request.base.sha }}
          head: ${{ github.event.pull_request.head.sha }}
          repo: ${{ github.repository }}
          review-id: github-pr-${{ github.event.pull_request.number }}
          feedback: balanced
          gate: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `install` | `true` | Install `@diffpal/diffpal` before review. |
| `diffpal-version` | `latest` | npm version or dist-tag for `@diffpal/diffpal`. Pin this for reproducible CI. |
| `diffpal-path` | `diffpal` | Path to an existing DiffPal binary. Custom paths skip automatic installation. |
| `base` | required | Base revision passed to `diffpal review github`. |
| `head` | required | Head revision passed to `diffpal review github`. |
| `config-dir` | empty | Extra config root directory. |
| `profile` | empty | DiffPal config profile. |
| `block-on` | `high` | Severity threshold that marks findings as blocking. |
| `gate` | `false` | Return non-zero when blocking findings exist. |
| `mode` | empty | Comma-separated GitHub publish modes. |
| `feedback` | `balanced` | Review feedback shape: `summary`, `balanced`, or `inline`. |
| `summary-overview` | `true` | Include a semantic change overview in summaries. |
| `out` | empty | Output findings bundle path. |
| `repo` | empty | Repository id for deterministic fingerprints. |
| `review-id` | empty | Review identifier for deterministic outputs. |
| `review-channel` | `diffpal` | Publishing channel for check runs and summary comments. |
| `max-files` | empty | Maximum files from diff. |
| `context-lines` | empty | Context lines to enrich each changed file. |
| `max-patch-chars` | empty | Maximum context characters per chunk. |
| `max-files-per-chunk` | empty | Maximum files per context chunk. |
| `language` | empty | Language for generated review findings. |
| `review-checks` | empty | Comma-separated checks, such as `security,bugs,performance,best-practices`. |
| `instructions` | empty | Additional review instructions for local prompt tuning. |
| `instructions-file` | empty | Path to additional review instructions. |

## Provider Setup

DiffPal delegates review work to the provider configured in your DiffPal config. Install and authenticate the provider CLI before this action step. The example above uses Codex ACP. Any ACP-compatible CLI can be used when your `.config/diffpal/config.yaml` points to it.

Full setup docs and CI examples live in the main repository: <https://github.com/diffpal/diffpal>.

## Permissions

Use these permissions when publishing PR feedback:

```yaml
permissions:
  contents: read
  pull-requests: write
  checks: write
```

Keep provider secrets available only to trusted workflows. For pull requests, use a same-repository guard when secrets are required.
