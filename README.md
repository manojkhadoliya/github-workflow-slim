# github-workflow-slim

Reference `.github` setup (styled after [school-framework](../school-framework)'s) plus a
self-auditing workflow that migrates eligible CI jobs from `ubuntu-latest` to the
1 vCPU / 5 GB, container-based [`ubuntu-slim`](https://github.com/actions/runner-images/blob/main/images/ubuntu-slim/ubuntu-slim-Readme.md)
runner using [`gh-slimify`](https://github.com/fchimpan/gh-slimify).

## Workflows

| File | Purpose | Expected slimify verdict |
|---|---|---|
| `ci.yml` | Install, lint, test, build | ✅ safe |
| `pr-title-lint.yml` | Conventional-commit PR title check | ✅ safe |
| `dependency-review.yml` | Flags newly-introduced vulnerable dependencies on PRs | ✅ safe |
| `stale.yml` | Marks stale issues/PRs on a schedule | ✅ safe |
| `release.yml` | Changesets-based version/publish | ⚠️ warning (long-running / npm publish) |
| `docker-build.yml` | Builds and pushes a Docker image | ❌ ineligible (Docker) |
| `integration-tests.yml` | Tests against a Postgres service container | ❌ ineligible (services) |
| `reusable-node-setup.yml` | `workflow_call` building block (checkout + Node + install) | n/a (no runner of its own) |
| `nightly-e2e.yml` | Thin shell calling the reusable workflow above | ❌ ineligible (calls a reusable workflow) |
| `slimify-audit.yml` | Scans all of the above with `gh-slimify` and opens a PR migrating the eligible jobs | — |

## Running the audit locally

```bash
gh extension install fchimpan/gh-slimify
gh slimify --all --json          # scan
gh slimify fix --all             # migrate the safe jobs in place
```

## `gh-slimify` command reference

Run all commands from the repo root (where `.github/workflows/` lives).

### Install

```bash
gh extension install fchimpan/gh-slimify
```

### Scan — read-only, never writes files or touches git

| Command | What it does |
|---|---|
| `gh slimify --help` | Show help |
| `gh slimify .github/workflows/ci.yml` | Scan one workflow file |
| `gh slimify .github/workflows/ci.yml .github/workflows/release.yml` | Scan multiple specific files |
| `gh slimify -f .github/workflows/ci.yml` (or `--file`) | Same, explicit flag form — repeatable |
| `gh slimify --all` | Scan every workflow in `.github/workflows/` |
| `gh slimify --all --skip-duration` | Skip the Actions-API run-duration lookup (avoids rate limits, faster) |
| `gh slimify --all --offline` | No API calls at all — durations report as unknown, Docker-action detection falls back to offline heuristics |
| `gh slimify --all --json` | Machine-readable JSON output (what `slimify-audit.yml` parses) |
| `gh slimify --all --verbose` | Add debug output for troubleshooting API/parsing issues |
| `gh slimify --all --json --skip-duration` | Combine: JSON output, no duration calls |
| `gh slimify --all --json --offline` | Combine: JSON output, fully offline |

### Fix — rewrites `runs-on:` in local files only; never commits, pushes, or opens a PR

| Command | What it does |
|---|---|
| `gh slimify fix .github/workflows/ci.yml` | Migrate eligible **safe** jobs in one file |
| `gh slimify fix --all` | Migrate eligible **safe** jobs across all workflows (default: warnings are skipped) |
| `gh slimify fix --all --force` | Also migrate **warning**-status jobs (missing commands, unknown duration, or 10-15m runtime) |
| `gh slimify fix --all --skip-duration` | Fix without the duration API call |
| `gh slimify fix --all --offline` | Fix fully offline |
| `gh slimify fix --all --json` | JSON output describing what was updated/skipped |
| `gh slimify fix .github/workflows/ci.yml --skip-duration --force` | Fix one file, no duration lookup, include warnings |

### Flag reference

| Flag | Scan | Fix | Meaning |
|---|---|---|---|
| `--all` | ✅ | ✅ | Target every file in `.github/workflows/` |
| `-f, --file <path>` | ✅ | ✅ | Target specific file(s); repeatable |
| `--json` | ✅ | ✅ | Machine-readable output |
| `--skip-duration` | ✅ | ✅ | Skip only the Actions-API duration check |
| `--offline` | ✅ | ✅ | Skip **all** API calls (durations + action metadata) |
| `--force` | — | ✅ | Also migrate `warning`-status jobs |
| `--verbose` | ✅ | ✅ | Debug logging |

### Migration status meaning

| Status | Recommended action | Trigger |
|---|---|---|
| `safe` | `migrate` | No missing commands, duration known and ≤ 10 min |
| `warning` | `review_before_migrate` | Missing commands, unknown duration, or 10-15 min duration |
| `ineligible` | `do_not_migrate` | Docker/`services:`/`container:`/privileged ops/reusable-workflow call/>15 min |
| `already_slim` | `no_action_needed` | Already `runs-on: ubuntu-slim` |
