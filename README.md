# SteadyCron GitHub Action

GitOps for cron jobs. Run a plan on every pull request and apply on merge to manage
cron jobs as code — reviewers see exactly what schedule and monitoring changes a PR
introduces, and the main branch is always the source of truth.

Supports two IaC tools:
- **`yaml`** (default) — declarative YAML manifest managed by the [SteadyCron CLI](https://github.com/steadycron/cli)
- **`terraform`** — HCL resources managed by the [SteadyCron Terraform provider](https://registry.terraform.io/providers/steadycron/steadycron/latest)

## Quick start — YAML manifest

**Step 1 — add your API key as a repository secret**

In your repository: **Settings → Secrets and variables → Actions → New repository secret**

| Name | Value |
|---|---|
| `STEADYCRON_API_KEY` | Your SteadyCron API key (`sc_…`) — create one under **Settings → API keys** |

**Step 2 — add two workflow files**

### Plan diff on pull requests

Posts a sticky comment showing exactly what cron changes a PR introduces. The check fails if the plan has errors, blocking the merge.

```yaml
# .github/workflows/cron-plan.yml
name: Cron plan

on:
  pull_request:
    paths:
      - 'steadycron.yaml'   # adjust to your manifest path

permissions:
  contents: read
  pull-requests: write      # required to post the plan comment

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: steadycron/action@v1
        with:
          command: plan
          manifest: steadycron.yaml
          comment-on-pr: 'true'
        env:
          STEADYCRON_API_KEY: ${{ secrets.STEADYCRON_API_KEY }}
```

### Apply on merge

Applies the manifest whenever the main branch is updated, keeping your account in sync with the repository.

```yaml
# .github/workflows/cron-apply.yml
name: Cron apply

on:
  push:
    branches:
      - main
    paths:
      - 'steadycron.yaml'   # adjust to your manifest path

permissions:
  contents: read

jobs:
  apply:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: steadycron/action@v1
        with:
          command: apply
          manifest: steadycron.yaml
          prune: 'true'
        env:
          STEADYCRON_API_KEY: ${{ secrets.STEADYCRON_API_KEY }}
```

## Quick start — Terraform

The same API key secret is used. No additional setup is needed beyond having your `.tf` files in the repository and a remote backend configured.

### Plan diff on pull requests

```yaml
# .github/workflows/cron-tf-plan.yml
name: Cron Terraform plan

on:
  pull_request:
    paths:
      - 'infra/steadycron/**'   # adjust to your .tf path

permissions:
  contents: read
  pull-requests: write          # required to post the plan comment

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: steadycron/action@v1
        with:
          tool: terraform
          command: plan
          working-directory: infra/steadycron
          comment-on-pr: 'true'
        env:
          STEADYCRON_API_KEY: ${{ secrets.STEADYCRON_API_KEY }}
```

### Apply on merge

```yaml
# .github/workflows/cron-tf-apply.yml
name: Cron Terraform apply

on:
  push:
    branches:
      - main
    paths:
      - 'infra/steadycron/**'   # adjust to your .tf path

permissions:
  contents: read

jobs:
  apply:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: steadycron/action@v1
        with:
          tool: terraform
          command: apply
          working-directory: infra/steadycron
        env:
          STEADYCRON_API_KEY: ${{ secrets.STEADYCRON_API_KEY }}
```

### Plan then apply in one job (with saved plan)

Running plan and apply as separate steps in the same job ensures apply uses the exact plan that was reviewed:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: steadycron/action@v1
        id: plan
        with:
          tool: terraform
          command: plan
          working-directory: infra/steadycron
        env:
          STEADYCRON_API_KEY: ${{ secrets.STEADYCRON_API_KEY }}

      - if: steps.plan.outputs.has-drift == 'true'
        uses: steadycron/action@v1
        with:
          tool: terraform
          command: apply
          working-directory: infra/steadycron
        env:
          STEADYCRON_API_KEY: ${{ secrets.STEADYCRON_API_KEY }}
```

When a `tfplan` file is present in `working-directory` from a prior plan step, `apply` uses it directly — no second plan is run, and no `-auto-approve` flag is needed.

### Remote state backend

Pass `-backend-config` lines (one per line) if your backend requires runtime configuration:

```yaml
- uses: steadycron/action@v1
  with:
    tool: terraform
    command: plan
    working-directory: infra/steadycron
    backend-config: |
      bucket=my-tfstate-bucket
      key=steadycron/terraform.tfstate
      region=eu-central-1
  env:
    STEADYCRON_API_KEY: ${{ secrets.STEADYCRON_API_KEY }}
    AWS_ACCESS_KEY_ID:  ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## Inputs

### Shared

| Input | Description | Default |
|---|---|---|
| `tool` | `yaml` or `terraform` | `yaml` |
| `command` | `plan` or `apply` (all tools); `validate`, `sync` (yaml only) | `plan` |
| `comment-on-pr` | Post the plan output as a sticky PR comment | `false` |

### YAML tool inputs

| Input | Description | Default |
|---|---|---|
| `manifest` | Path to the manifest file or directory | `.` |
| `prune` | Delete server resources not declared in the manifest | `false` |
| `cli-version` | Pin a specific CLI version (e.g. `1.3.0`) | `latest` |
| `env-file` | Path to a `.env` file for `${VAR}` substitution in the manifest | — |
| `namespace` | Account namespace (required when using `--prune` across multiple namespaces) | — |

### Terraform tool inputs

| Input | Description | Default |
|---|---|---|
| `working-directory` | Directory containing `.tf` files | `.` |
| `terraform-version` | Pin a specific Terraform version (e.g. `1.8.0`) | `latest` |
| `backend-config` | One or more `-backend-config=key=value` lines (newline-separated) | — |
| `var-file` | Path to a `.tfvars` file passed to plan/apply | — |

`STEADYCRON_API_KEY` is read from the step's `env:` block, not as an action input. This keeps the secret out of the action's inputs and follows GitHub's recommended pattern for sensitive values.

## Outputs

| Output | Description |
|---|---|
| `plan-json` | Raw plan JSON — `steadycron plan --output json` (yaml) or `terraform show -json` (terraform). Set when `command: plan`. |
| `plan-output` | Human-readable plan text (terraform only, set when `command: plan`) |
| `has-drift` | `"true"` when the plan detected changes |

Use outputs to gate downstream steps:

```yaml
- uses: steadycron/action@v1
  id: cron-plan
  with:
    tool: terraform          # or omit for yaml
    command: plan
    working-directory: infra/steadycron
  env:
    STEADYCRON_API_KEY: ${{ secrets.STEADYCRON_API_KEY }}

- if: steps.cron-plan.outputs.has-drift == 'true'
  run: echo "Drift detected — review the plan before merging."
```

## Manifest variable substitution

Any `${VAR}` placeholders in your manifest are resolved from the step's environment. Pass secrets alongside the API key:

```yaml
- uses: steadycron/action@v1
  with:
    command: apply
    manifest: steadycron.yaml
  env:
    STEADYCRON_API_KEY: ${{ secrets.STEADYCRON_API_KEY }}
    SLACK_WEBHOOK_URL:  ${{ secrets.SLACK_WEBHOOK_URL }}
```

Alternatively, point `env-file` at a `.env` file if you prefer to manage variables that way.

## Multiple environments

Use a matrix to manage separate staging and production manifests:

```yaml
jobs:
  apply:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        env: [staging, production]
    steps:
      - uses: actions/checkout@v4

      - uses: steadycron/action@v1
        with:
          command: apply
          manifest: manifests/${{ matrix.env }}.yaml
          prune: 'true'
        env:
          STEADYCRON_API_KEY: ${{ secrets[format('STEADYCRON_API_KEY_{0}', matrix.env)] }}
```

## Least-privilege API keys

For plan-only workflows (PR comments), create a **read-only** API key in the dashboard. It can run `plan` and `validate` but cannot apply changes — limiting the blast radius if the secret is ever compromised. Store it as a separate secret (e.g. `STEADYCRON_API_KEY_RO`).

## Supported platforms

Linux (x64, arm64) and macOS (x64, arm64). Windows runners are not supported.

## Links

- [Full CI/CD guide](https://steadycron.com/docs/ci-cd/)
- [Manifest reference](https://steadycron.com/docs/manifest-reference/)
- [IaC workflow](https://steadycron.com/docs/iac-workflow/)
- [SteadyCron CLI](https://github.com/steadycron/cli)
- [Terraform provider](https://github.com/steadycron/terraform-provider-steadycron)
- [Terraform Registry](https://registry.terraform.io/providers/steadycron/steadycron/latest)
