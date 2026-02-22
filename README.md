# Reusable GitHub Actions Workflows

Shared CI/CD workflows you can call from any repository.

## Available Workflows

| Workflow | File | Description |
|----------|------|-------------|
| [Security Scan](#security-scan) | `security-scan.yml` | Trivy, gosec, Bandit, npm audit with SARIF uploads and PR summary |
| [Terraform Plan](#terraform-plan) | `terraform-plan.yml` | Init, fmt, validate, plan with PR comment and cost estimate |
| [Terraform Destroy](#terraform-destroy) | `terraform-destroy.yml` | Plan destroy, manual approval gate, then destroy |
| [Go Tests](#go-tests) | `test-go.yml` | Go tests with race detector and coverage |
| [Node Tests](#node-tests) | `test-node.yml` | Node.js tests with npm |
| [Python Tests](#python-tests) | `test-python.yml` | Python tests with uv or pip |

## Prerequisites

- Calling repo must be in the same GitHub org, or this repo must be **public**
- For Terraform workflows: AWS OIDC role and (optionally) a Terraform Cloud token
- For security scan SARIF uploads: caller must set `permissions: security-events: write`

## Usage

### Security Scan

Runs Trivy filesystem scan, gosec (Go), Bandit (Python), and npm audit. Toggle each scanner on/off. Posts a summary table as a PR comment and uploads SARIF to the GitHub Security tab.

```yaml
name: CI
on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write
  security-events: write

jobs:
  security:
    uses: ShrithikShahapure/reuseable-actions/.github/workflows/security-scan.yml@main
    with:
      enable_trivy: true
      enable_gosec: true
      go_scan_path: './internal/...'
      enable_bandit: true
      python_scan_paths: 'src/ scripts/'
      enable_npm_audit: true
      node_working_directory: 'frontend'
      fail_on_critical: true
      post_pr_comment: true
```

**All inputs (with defaults):**

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `enable_trivy` | boolean | `true` | Run Trivy filesystem scan |
| `enable_gosec` | boolean | `false` | Run gosec |
| `go_scan_path` | string | `./...` | Go package path for gosec |
| `enable_bandit` | boolean | `false` | Run Bandit |
| `python_scan_paths` | string | `src/` | Paths for Bandit |
| `enable_npm_audit` | boolean | `false` | Run npm audit |
| `node_working_directory` | string | `.` | Directory for npm audit |
| `node_version` | string | `20` | Node.js version |
| `python_version` | string | `3.12` | Python version |
| `severity` | string | `CRITICAL,HIGH` | Trivy severity filter |
| `fail_on_critical` | boolean | `true` | Fail if CRITICALs found |
| `post_pr_comment` | boolean | `true` | Post summary to PR |

---

### Terraform Plan

Runs `terraform init`, `fmt -check`, `validate`, and `plan`. Posts plan diff and AWS cost estimate as PR comments.

```yaml
jobs:
  terraform:
    uses: ShrithikShahapure/reuseable-actions/.github/workflows/terraform-plan.yml@main
    with:
      terraform_dir: 'terraform'
      aws_region: 'us-east-1'
      aws_role_arn: 'arn:aws:iam::123456789012:role/github-actions-ci-role'
    secrets:
      TF_API_TOKEN: ${{ secrets.TF_API_TOKEN }}
```

**Inputs:**

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `terraform_dir` | string | `terraform` | Directory with `.tf` files |
| `aws_region` | string | `us-east-1` | AWS region |
| `aws_role_arn` | string | *required* | OIDC role ARN |
| `post_pr_comment` | boolean | `true` | Post plan to PR |
| `post_cost_estimate` | boolean | `true` | Post cost to PR |

**Secrets:**

| Secret | Required | Description |
|--------|----------|-------------|
| `TF_API_TOKEN` | No | Terraform Cloud API token |

---

### Terraform Destroy

Manual dispatch workflow with confirmation input, destroy plan in job summary, manual approval via GitHub issue, then destroy.

```yaml
name: Terraform Destroy
on:
  workflow_dispatch:
    inputs:
      confirm:
        description: 'Type "destroy" to confirm'
        type: string
        required: true

permissions:
  id-token: write
  contents: read
  issues: write

jobs:
  destroy:
    uses: ShrithikShahapure/reuseable-actions/.github/workflows/terraform-destroy.yml@main
    with:
      confirm: ${{ inputs.confirm }}
      terraform_dir: 'terraform'
      aws_region: 'us-east-1'
      aws_role_arn: 'arn:aws:iam::123456789012:role/github-actions-ci-role'
      approvers: 'your-github-username'
    secrets:
      TF_API_TOKEN: ${{ secrets.TF_API_TOKEN }}
```

**Inputs:**

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `confirm` | string | *required* | Must be `"destroy"` |
| `terraform_dir` | string | `terraform` | Directory with `.tf` files |
| `aws_region` | string | `us-east-1` | AWS region |
| `aws_role_arn` | string | *required* | OIDC role ARN |
| `approvers` | string | *required* | GitHub usernames (comma-separated) |

---

### Go Tests

```yaml
jobs:
  test-go:
    uses: ShrithikShahapure/reuseable-actions/.github/workflows/test-go.yml@main
    with:
      test_path: './internal/...'
      race: true
      coverage: true
```

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `test_path` | string | `./...` | Go package path |
| `race` | boolean | `true` | Enable `-race` |
| `coverage` | boolean | `true` | Generate coverage |
| `ref` | string | `''` | Git ref to checkout |

---

### Node Tests

```yaml
jobs:
  test-frontend:
    uses: ShrithikShahapure/reuseable-actions/.github/workflows/test-node.yml@main
    with:
      working_directory: 'frontend'
      node_version: '20'
      test_command: 'npm test'
```

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `working_directory` | string | `.` | Directory with `package.json` |
| `node_version` | string | `20` | Node.js version |
| `test_command` | string | `npm test` | Test command |
| `coverage` | boolean | `true` | Upload coverage artifacts |
| `ref` | string | `''` | Git ref to checkout |

---

### Python Tests

Supports both `uv` and `pip`. Optionally uses CPU-only PyTorch to save CI time.

```yaml
jobs:
  test-python:
    uses: ShrithikShahapure/reuseable-actions/.github/workflows/test-python.yml@main
    with:
      test_path: 'tests/unit/'
      package_manager: 'uv'
      cpu_only_torch: true
      artifact_name: 'pytest-unit'
```

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `test_path` | string | `tests/` | Test directory |
| `python_version` | string | `3.12` | Python version |
| `package_manager` | string | `uv` | `uv` or `pip` |
| `install_command` | string | `''` | Override install command |
| `test_command` | string | `''` | Override test command |
| `artifact_name` | string | `pytest-results` | Artifact name |
| `cpu_only_torch` | boolean | `false` | CPU-only PyTorch |
| `ref` | string | `''` | Git ref to checkout |

---

## Full Example: CI Pipeline

Here's a complete CI pipeline using all workflows together:

```yaml
name: CI
on:
  pull_request:
    branches: [main]

permissions:
  id-token: write
  contents: read
  pull-requests: write
  security-events: write

jobs:
  # Security scan runs first — mandatory gate
  security:
    uses: ShrithikShahapure/reuseable-actions/.github/workflows/security-scan.yml@main
    with:
      enable_trivy: true
      enable_gosec: true
      go_scan_path: './internal/...'
      enable_bandit: true
      python_scan_paths: 'src/ scripts/'
      enable_npm_audit: true
      node_working_directory: 'frontend'

  # Tests run after security scan passes
  test-go:
    needs: security
    uses: ShrithikShahapure/reuseable-actions/.github/workflows/test-go.yml@main
    with:
      test_path: './internal/...'

  test-frontend:
    needs: security
    uses: ShrithikShahapure/reuseable-actions/.github/workflows/test-node.yml@main
    with:
      working_directory: 'frontend'

  test-python:
    needs: security
    uses: ShrithikShahapure/reuseable-actions/.github/workflows/test-python.yml@main
    with:
      test_path: 'tests/unit/'
      cpu_only_torch: true

  # Terraform plan runs after tests pass
  terraform:
    needs: [test-go, test-frontend]
    uses: ShrithikShahapure/reuseable-actions/.github/workflows/terraform-plan.yml@main
    with:
      aws_role_arn: 'arn:aws:iam::123456789012:role/github-actions-ci-role'
    secrets:
      TF_API_TOKEN: ${{ secrets.TF_API_TOKEN }}
```

## Versioning

Pin to `@main` for latest, or use a specific commit SHA / tag for stability:

```yaml
uses: ShrithikShahapure/reuseable-actions/.github/workflows/security-scan.yml@v1.0.0
```
