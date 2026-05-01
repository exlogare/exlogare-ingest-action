# Exlogare Ingest Action

[![Marketplace](https://img.shields.io/badge/marketplace-exlogare--ingest--action-orange?logo=github)](https://github.com/marketplace/actions/exlogare-ingest)

Send failing CI logs from any GitHub Actions runner to
[Exlogare](https://exlogare.net) for AI root-cause analysis. The
action is a thin wrapper around the [`exl ingest`](https://github.com/exlogare/exlogare-cli)
CLI.

## When to use this action

You already have an Exlogare account and want to manually push logs
from a specific step (e.g. only the failed `pytest` job, not the
whole workflow). For zero-config tracking of every workflow_run
failure, install the
[Exlogare GitHub App](https://github.com/marketplace/exlogare) instead
— it ingests logs server-side without modifying your workflow YAML.

## Prerequisites

Before adding the action to a workflow, you need an Exlogare account and an
API token:

1. Sign up at [app.exlogare.net](https://app.exlogare.net) (free tier
   available — no credit card required).
2. Open **Settings → API tokens** and create a token with the `ingest` scope.
3. In your GitHub repository (or organization), add the token as a secret
   named `EXLOGARE_TOKEN` under **Settings → Secrets and variables →
   Actions**.

The action will not run without a valid token — the `ingest` endpoint rejects
unauthenticated requests with `401`.

## Quick start

```yaml
name: tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        id: tests
        run: pytest --tb=short | tee pytest.log
      - name: Send failure to Exlogare
        if: failure() && steps.tests.outcome == 'failure'
        uses: exlogare/exlogare-ingest-action@v1
        with:
          token: ${{ secrets.EXLOGARE_TOKEN }}
          log_path: pytest.log
```

## Inputs

| Name              | Required | Default                       | Description                                                                |
| ----------------- | -------- | ----------------------------- | -------------------------------------------------------------------------- |
| `token`           | yes      | —                             | Exlogare API token (use a secret).                                         |
| `log_path`        | no       | `''`                          | Path to a log file. Use `-` for stdin. Empty = auto-detect.                |
| `endpoint`        | no       | `https://api.exlogare.net`    | Override for self-hosted deployments.                                      |
| `fail_on_error`   | no       | `false`                       | Fail the step when ingest fails. Default keeps the original CI failure visible. |
| `cli_version`     | no       | `latest`                      | `exl` version (e.g. `0.2.1`) for deterministic builds.                     |

## Common patterns

### Send only the failed step's stdout

```yaml
- name: pytest
  id: tests
  run: pytest 2>&1 | tee pytest.log

- if: failure() && steps.tests.outcome == 'failure'
  uses: exlogare/exlogare-ingest-action@v1
  with:
    token: ${{ secrets.EXLOGARE_TOKEN }}
    log_path: pytest.log
```

### Read from stdin

```yaml
- if: failure()
  shell: bash
  env:
    EXLOGARE_TOKEN: ${{ secrets.EXLOGARE_TOKEN }}
  run: |
    cat pytest.log | npx exlogare/exlogare-ingest-action@v1 # NOT this
    # Composite actions can't take stdin from a previous step.
    # Use log_path: - and pipe inside the same step instead:
- if: failure()
  uses: exlogare/exlogare-ingest-action@v1
  with:
    token: ${{ secrets.EXLOGARE_TOKEN }}
    log_path: '-'
```

### Build matrix

The action is matrix-safe; each matrix leg ingests independently.

```yaml
strategy:
  matrix:
    py: ['3.10', '3.11', '3.12']
- name: pytest
  id: tests
  run: pytest 2>&1 | tee pytest-${{ matrix.py }}.log
- if: failure() && steps.tests.outcome == 'failure'
  uses: exlogare/exlogare-ingest-action@v1
  with:
    token: ${{ secrets.EXLOGARE_TOKEN }}
    log_path: pytest-${{ matrix.py }}.log
```

## Versioning

We follow [SemVer](https://semver.org/) and publish a rolling
`vMAJOR` tag (currently `v1`) that always points to the latest
`v1.x.y`. Pin to a full tag for reproducible builds.

```yaml
uses: exlogare/exlogare-ingest-action@v1        # latest v1.*
uses: exlogare/exlogare-ingest-action@v1.0.0    # pinned
```

## Troubleshooting

* **`401 unauthorized`** — Token expired or revoked. Issue a new one
  in the Exlogare dashboard.
* **`exl: command not found`** — The install step couldn't reach
  `https://exlogare.net/install.sh`. Check runner network egress; if
  GitHub-hosted runners can't reach exlogare.net (rare), the most
  likely cause is your repo's
  [outbound firewall settings](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners/networking).
* **Action runs but no analysis appears** — Check the run logs for
  `exlogare: ingest succeeded`. If absent, set `fail_on_error: true`
  to surface the underlying error.

## License

Apache-2.0. See [LICENSE](LICENSE).
