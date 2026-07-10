# DiffPal Review GitHub Action

GitHub Actions wrapper for DiffPal, the open-source, provider-agnostic AI review
system for pull requests.

The action installs the DiffPal CLI from npm by default, then runs
`diffpal review github` with the pull request base and head revisions. It gives
GitHub teams a portable AI review workflow without requiring a hosted DiffPal
service or per-seat review platform. Bring the provider recipe you want to use;
the review flow stays the same.

- Main DiffPal CLI repo: <https://github.com/diffpal/diffpal>
- GitHub Action package: `diffpal/action@v1`
- CLI package: <https://www.npmjs.com/package/@diffpal/diffpal>

## Quick Start

First generate and commit a DiffPal config:

```bash
npx -y @diffpal/diffpal@1.0.0 init --wizard --setup codex-api-key --platform github
```

This creates `.config/diffpal/config.yaml` with a visible `ci` profile. Existing
files are kept unless you pass `--force`.

Then add the workflow:

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
          feedback: review
          gate: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Inputs

Most repositories only need `base`, `head`, `profile`, `feedback`, and `gate`
from the quickstart. The rest are grouped by the `diffpal review github` flags
they configure.

### Action Install

| Input | Default | Use |
| --- | --- | --- |
| `install` | `true` | Install `@diffpal/diffpal` before review. |
| `diffpal-version` | `1` | npm version or dist-tag. Use an exact v1 release for reproducible CI. |
| `diffpal-path` | `diffpal` | Existing binary path. Custom paths skip install. |

### Config Selection

| Input | Default | CLI flag |
| --- | --- | --- |
| `config-dir` | empty | `--config-dir` |
| `profile` | empty | `--profile` |
| `debug` | `false` | `--debug` |

### Review Target

| Input | Default | CLI flag |
| --- | --- | --- |
| `base` | required | `--base` |
| `head` | required | `--head` |
| `repo` | empty | `--repo` |
| `review-id` | empty | `--review-id` |

### Publishing and Gate

| Input | Default | CLI flag |
| --- | --- | --- |
| `gate` | `false` | `--gate` |
| `block-on` | `high` | `--block-on` |
| `feedback` | `review` | `--feedback` |
| `summary-overview` | `true` | `--summary-overview` |
| `review-channel` | `diffpal` | `--review-channel` |
| `out` | empty | `--out` |

The exact `feedback` behavior is owned by the installed DiffPal CLI version.
Pin `diffpal-version` when you want rollout-safe, reproducible review behavior.

### Review Tuning

| Input | Default | CLI flag |
| --- | --- | --- |
| `language` | empty | `--language` |
| `instructions` | empty | `--instructions` |
| `instructions-file` | empty | `--instructions-file` |

## Provider Setup

DiffPal delegates review work to the provider configured in your DiffPal config.
Install and authenticate the provider CLI before this action step. The example
above uses Codex ACP. Any ACP-compatible CLI can be used when your
`.config/diffpal/config.yaml` points to it.

The onboarding wizard supports these setup recipes:

| Setup | Use when |
| --- | --- |
| `codex-api-key` | CI authenticates Codex with `OPENAI_API_KEY`. |
| `codex-subscription` | CI restores local Codex subscription auth. |
| `copilot-github-token` | CI authenticates Copilot with a fine-grained GitHub token. |
| `generic-acp` | You already have another ACP-compatible CLI. |

Full setup docs and CI examples live in the main repository:
<https://github.com/diffpal/diffpal>.

## Permissions

Use these permissions when publishing PR feedback:

```yaml
permissions:
  contents: read
  pull-requests: write
```

Keep provider secrets available only to trusted workflows. For pull requests,
use a same-repository guard when secrets are required.
