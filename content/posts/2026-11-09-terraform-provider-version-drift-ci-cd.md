---
title: "Terraform Provider Version Drift: Locking Down CI/CD Parity Before It Breaks Prod"
date: 2026-11-09
draft: true
tags: ["terraform", "cicd", "infrastructure-as-code", "dependency-management", "aws", "providers"]
---

## The problem

A new team member clones the repo, runs `terraform init`, and gets provider v5.2.0. Everything plans clean. CI runs the same config on a fresh runner, resolves v5.3.1 (which also satisfies the same loose `~> 5.0` constraint), and `terraform apply` fails on a breaking change in `aws_ebs_volume` tag handling that didn't exist in v5.2.0.

The drift isn't in your state. It's in provider version resolution itself.

This is the most insidious "works on my machine" failure mode in Terraform. It catches teams off guard because:

- The version constraint in `required_providers` looks reasonable (`~> 5.0`)
- `terraform plan` passes locally and in CI
- The failure shows up at apply time, not plan time, because the plan was generated against a different provider binary

A real example I keep coming back to: AWS provider v5.3.0 changed `aws_ebs_volume` `tags` from a map to a set of objects. A plan generated with v5.2.0 applies fine alongside v5.2.0. Apply the same plan with v5.3.1 and it fails with a schema validation error, because the provider now expects a different structure. Nothing about your Terraform code changed.

---

## Why this happens

Terraform's version constraint syntax (`~>`, `>=`, `!=`) is built for library dependency management, where semantic versioning (mostly) holds. Infrastructure providers play by different rules.

| Constraint | What it allows | The trap |
|------------|----------------|----------|
| `~> 5.0` | Any 5.x.y below 6.0 | Minor bumps can include breaking changes |
| `>= 4.0` | Any 4.x or 5.x and up | Major version jumps silently allowed |
| `!= 5.3.0` | Everything except one version | Whack-a-mole with every bad release |

Provider maintainers do ship breaking changes in patch and minor releases. The AWS changelog for v5.3.0 lists items as "bug fixes" that are really schema changes. The Azure provider hit similar issues in the v3.x line. Even HashiCorp's own providers aren't immune.

Without a committed lock file, every fresh `terraform init` (laptop, CI runner, ephemeral dev environment, GitHub Codespace) independently resolves the "latest compatible" version. Nothing coordinates them.

---

## Five design decisions, and the trade-offs

### 1. Dependency lock file strategy

**Option A: commit `terraform providers lock` to the repo (recommended)**

```bash
# Run once per provider upgrade, commit the result
terraform providers lock -platform=linux_amd64 -platform=darwin_arm64 -platform=windows_amd64
git add .terraform.lock.hcl
git commit -m "chore: update provider lock file for aws v5.2.0"
```

**Option B: generate the lock file in CI only**

```yaml
# .github/workflows/terraform.yml
- name: Generate lock file
  run: terraform providers lock -platform=linux_amd64
- name: Check for drift
  run: |
    if ! git diff --quiet .terraform.lock.hcl; then
      echo "Provider versions drifted! Commit the updated lock file."
      exit 1
    fi
```

**Decision: Option A.** Committing the lock file makes version resolution explicit and auditable. Every clone gets the exact same provider hashes. Option B is still valuable as a CI gate, but the source of truth should live in git, not in an ephemeral pipeline run.

---

### 2. Constraint tightening: exact vs. patch vs. minor

| Strategy | Constraint | Stability | Security updates | Maintenance burden |
|----------|------------|-----------|------------------|---------------------|
| **Exact pin** | `= 5.2.0` | Maximum | Manual only | High: every patch is a PR |
| **Patch range** | `~> 5.2.0` | High | Auto patch | Medium: minor bumps need review |
| **Minor range** | `~> 5.0` | Low | Auto minor | Low: but drift risk |

**My recommendation: patch-range (`~> 5.2.0`) plus a committed lock file.**

```hcl
# versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.2.0"  # Allow patches, block minors
    }
  }
}
```

**Why not exact?** Security fixes land in patch releases. Pin to `= 5.2.0` and you must manually upgrade for every CVE, and teams delay. `~> 5.2.0` still satisfies CVE-class patches while blocking the minor bump that could break your config.

**Why not minor-range?** The v5.3.0 breaking change above is the proof. `~> 5.0` lets that version through automatically. `~> 5.2.0` doesn't. You give up nothing you need: still get patches, still block the release that broke you.

---

### 3. CI enforcement: fail fast on drift

Add a dedicated job that only checks version resolution against the lock file:

```yaml
# .github/workflows/terraform-version-check.yml
name: Terraform Version Drift Check
on:
  pull_request:
  push:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # Weekly Monday 6 AM

jobs:
  version-drift:
    name: Check provider version drift
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.0

      - name: Init with lock file (fails if versions differ)
        run: |
          terraform init -backend=false
          # If .terraform.lock.hcl exists, init will use those exact versions
          # If resolution differs, Terraform exits non-zero

      - name: Verify no uncommitted lock file changes
        run: |
          if ! git diff --quiet .terraform.lock.hcl; then
            echo "::error::Provider lock file has uncommitted changes. Run 'terraform providers lock' locally and commit."
            git diff .terraform.lock.hcl
            exit 1
          fi
```

**The key insight:** `terraform init` with an existing `.terraform.lock.hcl` fails if the resolved versions don't match the lock file. That's your drift gate. It catches the problem at `init` time (seconds) instead of `apply` time (minutes or hours, after someone has already touched real infrastructure).

---

### 4. Provider upgrade workflow: automate the safe path

Don't make humans edit version pins by hand. Use Dependabot or Renovate with a staging validation gate:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "terraform"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "06:00"
    labels: ["terraform", "dependencies", "provider-upgrade"]
    # CRITICAL: Don't auto-merge. Require staging validation.
    allow:
      - dependency-type: "direct"
    ignore:
      # Block minor/major bumps; only let patches through as auto-PRs
      - dependency-name: "hashicorp/aws"
        versions: ["5.3.x", "6.x"]
```

The upgrade PR pipeline:

```yaml
# .github/workflows/provider-upgrade-validation.yml
name: Provider Upgrade Validation
on:
  pull_request:
    types: [opened, synchronize, reopened]
    labels: ["provider-upgrade"]

jobs:
  validate-in-staging:
    name: Validate provider upgrade in staging
    if: github.event.pull_request.labels.*.name contains 'provider-upgrade'
    runs-on: ubuntu-latest
    environment: staging  # Requires approval to proceed
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan (staging)
        id: plan
        run: terraform plan -var-file=staging.tfvars -out=tfplan -no-color

      - name: Terraform Apply (staging)
        if: github.event_name == 'pull_request' && github.event.action != 'closed'
        run: terraform apply -auto-approve tfplan

      - name: Run integration tests
        run: |
          # Put your smoke tests here, e.g. verify key resources exist
          aws ec2 describe-instances --filters "Name=tag:Environment,Values=staging"

      - name: Cleanup staging
        if: always()
        run: terraform destroy -auto-approve -var-file=staging.tfvars
```

**Why this works:** the upgrade PR proves the new provider version works in a real environment before it merges. The `environment: staging` gate forces manual approval (or a scheduled auto-approve for patch-only PRs), so a human signs off on the risk.

---

### 5. Multi-environment parity: dev, staging, and prod run identical providers

The lock file gives you version parity. You also need execution parity.

**The anti-pattern:** dev applies locally with v5.2.0 and CI uses v5.2.0, but someone ran `terraform init -upgrade` by hand on a staging bastion and it picked up v5.2.1. Now staging is the only environment you can't reproduce.

**Solution: a centralized provider cache plus immutable CI runners.**

```yaml
# .github/workflows/terraform-deploy.yml (shared workflow)
jobs:
  deploy:
    runs-on: ubuntu-latest
    # Use a pre-baked runner image with providers already installed
    container:
      image: ghcr.io/yourorg/terraform-runner:v1.9.0-aws-5.2.0
    steps:
      - uses: actions/checkout@v4
      # No setup-terraform needed: providers are pre-installed
      - run: terraform init -plugin-dir=/opt/terraform/plugins
      - run: terraform plan -out=tfplan
      - run: terraform apply tfplan
```

**Dockerfile for the runner image:**

```dockerfile
FROM hashicorp/terraform:1.9.0

# Pre-install exact provider versions from the lock file
ARG AWS_PROVIDER_VERSION=5.2.0
RUN mkdir -p /opt/terraform/plugins/linux_amd64 && \
    curl -sL "https://releases.hashicorp.com/terraform-provider-aws/${AWS_PROVIDER_VERSION}/terraform-provider-aws_${AWS_PROVIDER_VERSION}_linux_amd64.zip" \
    -o /tmp/aws.zip && \
    unzip -o /tmp/aws.zip -d /opt/terraform/plugins/linux_amd64/ && \
    rm /tmp/aws.zip

ENTRYPOINT ["terraform"]
```

**Local dev parity:** give developers a `devcontainer.json` or `mise.toml` that pins the same versions:

```toml
# .mise.toml (or .tool-versions for asdf)
[tools]
terraform = "1.9.0"
# Provider versions managed via .terraform.lock.hcl
```

---

## Decision matrix: when to pin what

| Scenario | Constraint | Lock file | CI gate | Upgrade process |
|----------|------------|-----------|---------|-----------------|
| **Production-critical (prod DB, networking)** | `= 5.2.0` (exact) | Committed | Block on drift | Manual PR + full staging validation |
| **Standard workloads (compute, storage)** | `~> 5.2.0` (patch) | Committed | Block on drift | Dependabot weekly + staging validation |
| **Experimental / dev-only modules** | `~> 5.0` (minor) | Committed | Warn only | Auto-merge patches, manual for minors |
| **Shared provider modules (org registry)** | `~> 5.2.0` | Committed | Block on drift | Semantic release with integration tests |

**Rule of thumb:** the closer a workload sits to production data or availability, the tighter the pin. But always commit the lock file. It's your source of truth no matter which constraint strategy you pick.

---

## The master checklist: implement this tomorrow

1. **Generate and commit the lock file** (one-time run):

   ```bash
   terraform providers lock -platform=linux_amd64 -platform=darwin_arm64 -platform=windows_amd64
   git add .terraform.lock.hcl
   git commit -m "chore: add provider lock file for reproducible builds"
   ```

2. **Tighten constraints to patch-range** in `versions.tf`:

   ```hcl
   version = "~> 5.2.0"  # Not "~> 5.0"
   ```

3. **Add the drift-check workflow** (`.github/workflows/terraform-version-check.yml`). This is your safety net.

4. **Configure Dependabot** for weekly patch-only PRs with the `provider-upgrade` label.

5. **Build a validated runner image** with providers pre-installed. It removes "but it works on my machine" at the infrastructure level, not just the config level.

6. **Document the upgrade runbook** as a one-pager in `docs/provider-upgrades.md` covering:
   - How to test a minor bump locally
   - How to approve the staging validation
   - The rollback procedure if the staging apply fails

---

## Summary

Provider version drift is a coordination problem, not a Terraform bug. The tools to solve it have existed since Terraform 0.14 (`terraform providers lock`), but most teams never use them systematically.

The pattern that works:

- **Lock file in git** as the single source of truth
- **Patch-range constraints** for security updates without surprise breaking changes
- **A CI gate at `init` time** to catch drift before it wastes compute or touches state
- **Staging-validated upgrades** so prod only sees versions that have already proven themselves
- **Immutable runner images** so dev, staging, and prod execute identical binaries

You don't need all five on day one. Start with #1 and #3: commit the lock file and add the drift check. That alone eliminates most of the "works on my machine" failures. The rest is progressive hardening as your team and infrastructure grow.

The goal isn't to never upgrade providers. It's to upgrade on your terms, with evidence, not hope.

---

*I've published a reference implementation with working GitHub Actions, Dependabot config, and the validated runner Dockerfile, available on request if you'd like the complete templates.*