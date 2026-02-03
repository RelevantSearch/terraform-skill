# CI/CD Workflows for Terraform

> **Part of:** [rs-terraform-skill](../SKILL.md)
> **Purpose:** CI/CD integration patterns for Terraform/OpenTofu

This document provides detailed CI/CD workflow templates and optimization strategies for infrastructure-as-code pipelines.

---

## Table of Contents

1. [rs_infra CI/CD Workflows](#rs_infra-cicd-workflows)
2. [Workload Identity Federation](#workload-identity-federation)
3. [GitHub Actions Workflow](#github-actions-workflow)
4. [GitLab CI Template](#gitlab-ci-template)
5. [Cost Optimization](#cost-optimization)
6. [Automated Cleanup](#automated-cleanup)
7. [Best Practices](#best-practices)

---

## rs_infra CI/CD Workflows

The rs_infra codebase uses GitHub Actions with these workflows:

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `terragrunt-plan` | PR | Lint, plan, security scan |
| `terragrunt-apply` | Push to main | Apply changes |
| `terraform-docs` | Module changes | Update READMEs |
| `drift-detection` | Daily 6 AM | Detect configuration drift |

### terragrunt-plan.yml

```yaml
name: Terragrunt Plan

on:
  pull_request:
    paths:
      - 'environments/**'
      - 'modules/**'

permissions:
  contents: read
  id-token: write  # Required for WIF
  pull-requests: write

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.WIF_PROVIDER }}
          service_account: ${{ vars.WIF_SERVICE_ACCOUNT }}

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: 1.8.8

      - name: Setup Terragrunt
        run: |
          curl -L -o terragrunt https://github.com/gruntwork-io/terragrunt/releases/download/v0.72.6/terragrunt_linux_amd64
          chmod +x terragrunt && sudo mv terragrunt /usr/local/bin/

      - name: Format Check
        run: tofu fmt -recursive -check

      - name: Lint
        uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: latest
      - run: tflint --recursive

      - name: Security Scan
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          config_file: .checkov.yaml

      - name: Terragrunt Plan
        run: |
          cd environments
          terragrunt run-all plan --terragrunt-non-interactive

      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            // Post plan summary to PR
```

### terragrunt-apply.yml

```yaml
name: Terragrunt Apply

on:
  push:
    branches: [main]
    paths:
      - 'environments/**'
      - 'modules/**'

permissions:
  contents: read
  id-token: write

jobs:
  apply:
    runs-on: ubuntu-latest
    environment: production  # Requires approval
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.WIF_PROVIDER }}
          service_account: ${{ vars.WIF_SERVICE_ACCOUNT }}

      - name: Setup Tools
        # ... same as plan

      - name: Terragrunt Apply
        run: |
          cd environments
          terragrunt run-all apply --terragrunt-non-interactive
```

---

## Workload Identity Federation

Per ADR-008, use WIF instead of service account keys:

### Setup

1. **Create Workload Identity Pool**:
```hcl
resource "google_iam_workload_identity_pool" "github" {
  project                   = var.project_id
  workload_identity_pool_id = "github-pool"
  display_name              = "GitHub Actions Pool"
}
```

2. **Create Provider**:
```hcl
resource "google_iam_workload_identity_pool_provider" "github" {
  project                            = var.project_id
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-provider"

  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.actor"      = "assertion.actor"
    "attribute.repository" = "assertion.repository"
  }

  attribute_condition = "assertion.repository == 'RelevantSearch/rs_infra'"

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }
}
```

3. **Grant Access**:
```hcl
resource "google_service_account_iam_binding" "workload_identity" {
  service_account_id = google_service_account.ci.name
  role               = "roles/iam.workloadIdentityUser"

  members = [
    "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.repository/RelevantSearch/rs_infra"
  ]
}
```

### GitHub Actions Usage

```yaml
- name: Authenticate to GCP
  uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/123456/locations/global/workloadIdentityPools/github-pool/providers/github-provider
    service_account: ci-runner@rs-infra-484217.iam.gserviceaccount.com
```

**Benefits:**
- No long-lived credentials
- Automatic token rotation
- Repository-scoped access
- Audit trail in GCP

---

## GitHub Actions Workflow

### Complete Example

```yaml
# .github/workflows/terraform.yml
name: Terraform

on: [push, pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: hashicorp/setup-terraform@v2

      - name: Terraform Format
        run: terraform fmt -check -recursive

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: TFLint
        run: |
          curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
          tflint --init
          tflint

  test:
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run Terraform Tests
        run: terraform test

      # Or for Terratest:
      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'

      - name: Run Terratest
        run: |
          cd tests
          go test -v -timeout 30m -parallel 4

  plan:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: hashicorp/setup-terraform@v2

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan -out=tfplan

      - name: Upload Plan
        uses: actions/upload-artifact@v3
        with:
          name: tfplan
          path: tfplan

  apply:
    needs: plan
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production
    steps:
      - uses: actions/checkout@v3
      - uses: hashicorp/setup-terraform@v2

      - name: Download Plan
        uses: actions/download-artifact@v3
        with:
          name: tfplan

      - name: Terraform Apply
        run: terraform apply tfplan
```

### With Cost Estimation (Infracost)

```yaml
  cost-estimate:
    needs: plan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Infracost
        uses: infracost/actions/setup@v2
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Generate Cost Estimate
        run: |
          infracost breakdown --path . \
            --format json \
            --out-file /tmp/infracost.json

      - name: Post Cost Comment
        uses: infracost/actions/comment@v1
        with:
          path: /tmp/infracost.json
          behavior: update
```

---

## GitLab CI Template

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - test
  - plan
  - apply

variables:
  TF_ROOT: ${CI_PROJECT_DIR}

.terraform_template:
  image: hashicorp/terraform:latest
  before_script:
    - cd ${TF_ROOT}
    - terraform init

validate:
  extends: .terraform_template
  stage: validate
  script:
    - terraform fmt -check -recursive
    - terraform validate

test:
  extends: .terraform_template
  stage: test
  script:
    - terraform test
  only:
    - merge_requests
    - main

plan:
  extends: .terraform_template
  stage: plan
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 1 week
  only:
    - merge_requests
    - main

apply:
  extends: .terraform_template
  stage: apply
  script:
    - terraform apply tfplan
  dependencies:
    - plan
  only:
    - main
  when: manual
  environment:
    name: production
```

---

## Cost Optimization

### Strategy

1. **Use mocking for PR validation** (free)
2. **Run integration tests only on main branch** (controlled cost)
3. **Implement auto-cleanup** (prevent orphaned resources)
4. **Tag all test resources** (track spending)

### Example: Conditional Test Execution

```yaml
# GitHub Actions
test:
  runs-on: ubuntu-latest
  steps:
    - name: Run Unit Tests (Mocked)
      run: terraform test

    - name: Run Integration Tests
      if: github.ref == 'refs/heads/main'
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      run: |
        cd tests
        go test -v -timeout 30m
```

### Cost-Aware Test Tags

```go
// In Terratest
terraformOptions := &terraform.Options{
    TerraformDir: "../examples/complete",
    Vars: map[string]interface{}{
        "tags": map[string]string{
            "Environment": "test",
            "TTL":         "2h",
            "CreatedBy":   "CI",
            "JobID":       os.Getenv("GITHUB_RUN_ID"),
        },
    },
}
```

---

## Automated Cleanup

### Cleanup Script (Bash)

```bash
#!/bin/bash
# cleanup-test-resources.sh

# Find and terminate instances older than 2 hours with test tag
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Environment,Values=test \
  --query 'ResourceTagMappingList[?Tags[?Key==`TTL` && Value<`'$(date -u -d '2 hours ago' +%Y-%m-%dT%H:%M:%S)'`]].ResourceARN' \
  --output text | \
  while read arn; do
    instance_id=$(echo $arn | grep -oP 'instance/\K[^/]+')
    if [ ! -z "$instance_id" ]; then
      echo "Terminating instance: $instance_id"
      aws ec2 terminate-instances --instance-ids $instance_id
    fi
  done
```

### Scheduled Cleanup (GitHub Actions)

```yaml
# .github/workflows/cleanup.yml
name: Cleanup Test Resources

on:
  schedule:
    - cron: '0 */2 * * *'  # Every 2 hours
  workflow_dispatch:        # Manual trigger

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Run Cleanup Script
        run: ./scripts/cleanup-test-resources.sh
```

---

## Best Practices

### 1. Separate Environments

```yaml
# Different workflows for different environments
.github/workflows/
  terraform-dev.yml
  terraform-staging.yml
  terraform-prod.yml
```

Or use reusable workflows:

```yaml
# .github/workflows/terraform-deploy.yml (reusable)
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string

jobs:
  deploy:
    environment: ${{ inputs.environment }}
    # ... deployment steps
```

### 2. Require Approvals for Production

```yaml
apply:
  environment:
    name: production
    # Requires manual approval in GitHub
  when: manual
```

### 3. Use Remote State

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

### 4. Implement State Locking

```yaml
# In CI, use -lock-timeout to handle concurrent runs
- name: Terraform Apply
  run: terraform apply -lock-timeout=10m tfplan
```

### 5. Cache Terraform Plugins

```yaml
# GitHub Actions
- name: Cache Terraform Plugins
  uses: actions/cache@v3
  with:
    path: |
      ~/.terraform.d/plugin-cache
    key: ${{ runner.os }}-terraform-${{ hashFiles('**/.terraform.lock.hcl') }}
```

### 6. Security Scanning in CI

```yaml
security-scan:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v3

    - name: Run Trivy
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'config'
        scan-ref: '.'

    - name: Run Checkov
      uses: bridgecrewio/checkov-action@master
      with:
        directory: .
        framework: terraform
```

---

## Atlantis Integration

[Atlantis](https://www.runatlantis.io/) provides Terraform automation via pull request comments.

### atlantis.yaml

```yaml
version: 3
projects:
  - name: production
    dir: environments/prod
    workspace: default
    terraform_version: v1.6.0
    workflow: custom

workflows:
  custom:
    plan:
      steps:
        - init
        - plan:
            extra_args: ["-lock", "false"]
    apply:
      steps:
        - apply
```

### Benefits

- Plan results as PR comments
- Apply via PR comments
- Locking prevents concurrent changes
- Integrates with VCS (GitHub, GitLab, Bitbucket)

---

## Troubleshooting

### Issue: Tests fail in CI but pass locally

**Cause:** Different Terraform/provider versions

**Solution:**

```hcl
# versions.tf - Pin versions
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

### Issue: Parallel tests conflict

**Cause:** Resource naming collisions

**Solution:**

```go
// Use unique identifiers
uniqueId := random.UniqueId()
bucketName := fmt.Sprintf("test-bucket-%s-%s",
    os.Getenv("GITHUB_RUN_ID"),
    uniqueId)
```

---

**Back to:** [Main Skill File](../SKILL.md)
