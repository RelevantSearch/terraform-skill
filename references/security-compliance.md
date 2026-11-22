# Security & Compliance

> **Part of:** [rs-terraform-skill](../SKILL.md)
> **Purpose:** Security best practices and compliance patterns for Terraform/OpenTofu

This document provides security hardening guidance and compliance automation strategies for infrastructure-as-code.

---

## Table of Contents

1. [rs_infra Security Patterns](#rs_infra-security-patterns)
2. [Security Scanning Tools](#security-scanning-tools)
3. [Common Security Issues](#common-security-issues)
4. [Compliance Testing](#compliance-testing)
5. [Secrets Management](#secrets-management)
6. [State File Security](#state-file-security)

---

## rs_infra Security Patterns

The rs_infra codebase implements several security patterns documented in ADRs:

### CMEK Encryption (ADR-003)

All data at rest must use Customer-Managed Encryption Keys:

```hcl
# Pattern: Cloud KMS key with autokey
resource "google_kms_key_ring" "this" {
  name     = "${var.project_name}-keyring"
  location = var.region
  project  = var.project_id
}

resource "google_kms_crypto_key" "this" {
  name            = "${var.project_name}-key"
  key_ring        = google_kms_key_ring.this.id
  rotation_period = "7776000s"  # 90 days

  lifecycle {
    prevent_destroy = true
  }
}

# Usage in storage resources
resource "google_storage_bucket" "this" {
  name     = var.bucket_name
  location = var.region

  encryption {
    default_kms_key_name = google_kms_crypto_key.this.id
  }
}
```

### Group-Based IAM (ADR-004)

Use Google Groups for IAM instead of individual users:

```hcl
# Pattern: IAM binding to Google Group
resource "google_project_iam_member" "this" {
  project = var.project_id
  role    = "roles/viewer"
  member  = "group:${var.group_email}"  # e.g., "developers@relevantsearch.com"
}

# Anti-pattern: Individual user bindings
# resource "google_project_iam_member" "bad" {
#   member = "user:john@example.com"  # DON'T DO THIS
# }
```

### Organization Policies (ADR-006)

Enforce security policies at organization level:

```hcl
# Pattern: Disable default service account creation
resource "google_org_policy_policy" "disable_service_account_creation" {
  name   = "organizations/${var.org_id}/policies/iam.disableServiceAccountCreation"
  parent = "organizations/${var.org_id}"

  spec {
    rules {
      enforce = "TRUE"
    }
  }
}

# Pattern: Restrict VM external IP addresses
resource "google_org_policy_policy" "vm_external_ip" {
  name   = "organizations/${var.org_id}/policies/compute.vmExternalIpAccess"
  parent = "organizations/${var.org_id}"

  spec {
    rules {
      deny_all = "TRUE"
    }
  }
}
```

### Checkov Skip Documentation

When security checks must be skipped, document in `.checkov.yaml`:

```yaml
# .checkov.yaml
skip-check:
  # CKV_GCP_XX: Reason for skipping, reference ADR if applicable
  - CKV_GCP_62  # Skip: GKE private cluster requires specific network setup per ADR-001
```

**Always include:**
1. The check ID
2. A clear reason
3. Reference to ADR if architectural decision

### Workload Identity Federation (ADR-008)

For CI/CD pipelines, use WIF instead of service account keys:

```hcl
# Pattern: Workload Identity Pool for GitHub Actions
resource "google_iam_workload_identity_pool" "github" {
  project                   = var.project_id
  workload_identity_pool_id = "github-pool"
  display_name              = "GitHub Actions Pool"
}

resource "google_iam_workload_identity_pool_provider" "github" {
  project                            = var.project_id
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-provider"
  display_name                       = "GitHub Provider"

  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.actor"      = "assertion.actor"
    "attribute.repository" = "assertion.repository"
  }

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }
}
```

---

## Security Scanning Tools

### Essential Security Checks

```bash
# Static security scanning
trivy config .
checkov -d .

# Compliance testing
terraform-compliance -f compliance/ -p tfplan.json
```

### Trivy Integration

**Install:**

```bash
# macOS
brew install trivy

# Linux
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# In CI
- uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'config'
    scan-ref: '.'
```

**Note:** Trivy is the successor to tfsec, maintained by Aqua Security.

**Example Output:**

```
Result #1 HIGH Security group rule allows egress to multiple public internet addresses
────────────────────────────────────────────────────────────────────────────────
  security.tf:15-20

   12 | resource "aws_security_group_rule" "egress" {
   13 |   type              = "egress"
   14 |   from_port         = 0
   15 |   to_port           = 0
   16 |   protocol          = "-1"
   17 |   cidr_blocks       = ["0.0.0.0/0"]
   18 |   security_group_id = aws_security_group.this.id
   19 | }
```

### Checkov Integration

```bash
# Run Checkov
checkov -d . --framework terraform

# Skip specific checks
checkov -d . --skip-check CKV_AWS_23

# Generate JSON report
checkov -d . -o json > checkov-report.json
```

---

## Common Security Issues

### ❌ DON'T: Store Secrets in Variables

```hcl
# BAD: Secret in plaintext
variable "database_password" {
  type    = string
  default = "SuperSecret123!"  # ❌ Never do this
}
```

### ✅ DO: Use Secrets Manager

```hcl
# Good: Reference secrets from AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/database/password"
}

resource "aws_db_instance" "this" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

### ❌ DON'T: Use Default VPC

```hcl
# BAD: Default VPC has public subnets
resource "aws_instance" "app" {
  ami           = "ami-12345"
  subnet_id     = "subnet-default"  # ❌ Avoid default resources
}
```

### ✅ DO: Create Dedicated VPCs

```hcl
# Good: Custom VPC with private subnets
resource "aws_vpc" "this" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
}

resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.this.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
}
```

### ❌ DON'T: Skip Encryption

```hcl
# BAD: Unencrypted S3 bucket
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
  # ❌ No encryption configured
}
```

### ✅ DO: Enable Encryption at Rest

```hcl
# Good: Enable encryption
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

### ❌ DON'T: Open Security Groups to Internet

```hcl
# BAD: Security group open to internet
resource "aws_security_group_rule" "allow_all" {
  type              = "ingress"
  from_port         = 0
  to_port           = 65535
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]  # ❌ Never do this
  security_group_id = aws_security_group.this.id
}
```

### ✅ DO: Use Least-Privilege Security Groups

```hcl
# Good: Restrict to specific ports and sources
resource "aws_security_group_rule" "app_https" {
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = ["10.0.0.0/16"]  # ✅ Internal only
  security_group_id = aws_security_group.this.id
}
```

---

## Compliance Testing

### terraform-compliance

**Install:**

```bash
pip install terraform-compliance
```

**Example Compliance Test:**

```gherkin
# compliance/aws-encryption.feature
Feature: AWS Resources must be encrypted

  Scenario: S3 buckets must have encryption
    Given I have aws_s3_bucket defined
    When it has aws_s3_bucket_server_side_encryption_configuration
    Then it must contain rule
    And it must contain apply_server_side_encryption_by_default

  Scenario: RDS instances must be encrypted
    Given I have aws_db_instance defined
    Then it must contain storage_encrypted
    And its value must be true
```

**Run Tests:**

```bash
# Generate plan in JSON
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json

# Run compliance tests
terraform-compliance -f compliance/ -p tfplan.json
```

### Open Policy Agent (OPA)

```rego
# policy/s3_encryption.rego
package terraform.s3

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  not resource.change.after.server_side_encryption_configuration

  msg := sprintf("S3 bucket '%s' must have encryption enabled", [resource.address])
}
```

---

## Secrets Management

### AWS Secrets Manager Pattern

```hcl
# Create secret
resource "aws_secretsmanager_secret" "db_password" {
  name        = "prod/database/password"
  description = "RDS master password"

  recovery_window_in_days = 30
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = random_password.db_password.result
}

# Generate secure password
resource "random_password" "db_password" {
  length  = 32
  special = true
}

# Use secret in RDS
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db_password.id
}

resource "aws_db_instance" "this" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
  # ...
}
```

### Environment Variables

```bash
# Never commit these
export TF_VAR_database_password="secret123"
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

**In .gitignore:**

```
*.tfvars
.env
secrets/
```

---

## State File Security

### Encrypt State at Rest

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true  # ✅ Always enable encryption
  }
}
```

### Secure State Bucket

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state"
}

# Enable versioning (protect against accidental deletion)
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Enable encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Restrict State Access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/TerraformRole"
      },
      "Action": [
        "s3:ListBucket",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-terraform-state",
        "arn:aws:s3:::my-terraform-state/*"
      ]
    }
  ]
}
```

---

## IAM Best Practices

### ✅ DO: Use Least Privilege

```hcl
# Good: Specific permissions only
resource "aws_iam_policy" "app_policy" {
  name = "app-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "arn:aws:s3:::my-app-bucket/*"
      }
    ]
  })
}
```

### ❌ DON'T: Use Wildcard Permissions

```hcl
# BAD: Overly broad permissions
resource "aws_iam_policy" "bad_policy" {
  policy = jsonencode({
    Statement = [
      {
        Effect   = "Allow"
        Action   = "*"  # ❌ Never use wildcard
        Resource = "*"
      }
    ]
  })
}
```

---

## Compliance Checklists

### SOC 2 Compliance

- [ ] Encryption at rest for all data stores
- [ ] Encryption in transit (TLS/SSL)
- [ ] IAM policies follow least privilege
- [ ] Logging enabled for all resources
- [ ] MFA required for privileged access
- [ ] Regular security scanning in CI/CD

### HIPAA Compliance

- [ ] PHI encrypted at rest and in transit
- [ ] Access logs enabled
- [ ] Dedicated VPC with private subnets
- [ ] Regular backup and retention policies
- [ ] Audit trail for all infrastructure changes

### PCI-DSS Compliance

- [ ] Network segmentation (separate VPCs)
- [ ] No default passwords
- [ ] Strong encryption algorithms
- [ ] Regular security scanning
- [ ] Access control and monitoring

---

## Resources

- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [Checkov Documentation](https://www.checkov.io/)
- [terraform-compliance](https://terraform-compliance.com/)
- [Open Policy Agent](https://www.openpolicyagent.org/)
- [AWS Security Best Practices](https://aws.amazon.com/security/best-practices/)

---

**Back to:** [Main Skill File](../SKILL.md)
