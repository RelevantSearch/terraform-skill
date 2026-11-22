# Terraform Skill Modification Plan

**Purpose:** Adapt terraform-skill to better support the rs_infra repository and its agentic development workflow.

**Related Documents:**
- [ADR-009: Test Coverage Strategy](../infra_agent_flow/docs/adr/ADR-009-test-coverage-strategy.md)
- [ADR-010: Agentic Development Workflow](../infra_agent_flow/docs/adr/ADR-010-agentic-development-workflow.md)
- [Agentic Workflow Design](../infra_agent_flow/docs/design/agentic-workflow-design.md)

---

## 1. Features to Preserve

These best practices from the original terraform-skill should be retained:

### Architecture & Structure
- [ ] **Progressive Disclosure** - Core SKILL.md + reference files for on-demand loading
- [ ] **Token Efficiency** - Keep under ~30K tokens total, <500 lines for SKILL.md
- [ ] **Decision Matrices** - "When to use" guidance with clear criteria

### Content Quality
- [ ] **Imperative Voice** - "Use X" not "You should consider X"
- [ ] **✅/❌ Patterns** - Side-by-side DO vs DON'T examples
- [ ] **Concrete Examples** - Complete, runnable code with inline comments
- [ ] **Version Requirements** - Clearly marked for each feature

### TDD Framework
- [ ] **Baseline Scenarios** - RED phase testing without skill
- [ ] **Compliance Verification** - GREEN phase with skill
- [ ] **Rationalization Table** - Counter common agent excuses

### Automation
- [ ] **Conventional Commits** - Automatic versioning (feat!/feat/fix)
- [ ] **Validation Workflow** - Frontmatter, links, markdown linting
- [ ] **Release Automation** - Version sync across marketplace.json, SKILL.md

---

## 2. Modifications for rs_infra Alignment

### 2.1 Testing Framework Updates

**Current State (terraform-skill):**
- Generic testing pyramid (static → integration → E2E)
- Terratest-focused with OpenTofu 1.6+ native tests mentioned

**Target State (rs_infra aligned):**
- Primary: OpenTofu native tests (`.tftest.hcl`) for unit testing
- Secondary: Terratest (Go) for integration testing
- Established patterns from `tests/helpers/` library

**Changes to `references/testing-frameworks.md`:**

```markdown
## rs_infra Testing Strategy (ADR-009)

### Test Types

| Type | Framework | Location | When to Use |
|------|-----------|----------|-------------|
| Unit | OpenTofu native | `modules/*/tests/*.tftest.hcl` | Variable validation, output verification |
| Integration | Terratest (Go) | `tests/integration/*.go` | Real GCP resource validation |

### Unit Test Pattern (OpenTofu Native)

```hcl
# modules/<module>/tests/valid_inputs.tftest.hcl

mock_provider "google" {}
mock_provider "google-beta" {}

variables {
  project_id = "test-project"
  name       = "test-resource"
}

run "valid_configuration" {
  command = plan

  assert {
    condition     = var.name == "test-resource"
    error_message = "Name should be set correctly"
  }
}
```

### Integration Test Pattern (Terratest)

```go
// tests/integration/module_test.go
package integration

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "rs_infra/tests/helpers"
)

func TestModuleCreatesResources(t *testing.T) {
    t.Parallel()

    projectID := helpers.GetTestProjectID(t)
    uniqueID := helpers.UniqueID()

    options := helpers.TerraformOptions(t, "../../modules/<module>", map[string]interface{}{
        "project_id": projectID,
        "name":       fmt.Sprintf("test-%s", uniqueID),
    })

    defer terraform.Destroy(t, options)
    terraform.InitAndApply(t, options)

    // Validate with GCP API
    helpers.ValidateResource(t, projectID, "resource-name")
}
```

### Test Execution

```bash
# Unit tests (fast, no resources)
tofu -chdir=modules/<module> init -backend=false
tofu -chdir=modules/<module> test

# Integration tests (requires TEST_PROJECT_ID)
cd tests && go test -v -timeout 30m ./integration/...
```
```

---

### 2.2 Module Structure Alignment

**Current State (terraform-skill):**
- Generic module structure with examples/, tests/

**Target State (rs_infra aligned):**
- Specific structure matching rs_infra patterns
- Terragrunt integration for state management
- Environment-based organization

**Changes to `references/module-patterns.md`:**

```markdown
## rs_infra Module Structure

### Standard Module Layout

```
modules/
└── <module-name>/
    ├── main.tf              # Resource definitions
    ├── variables.tf         # Input variables with descriptions
    ├── outputs.tf           # Output values with descriptions
    ├── versions.tf          # Provider version constraints
    ├── README.md            # Auto-generated via terraform-docs
    └── tests/
        └── valid_inputs.tftest.hcl  # OpenTofu native tests
```

### Environment Structure (Terragrunt)

```
environments/
├── _env/                    # Shared environment includes
│   └── testing.hcl
├── common/                  # Shared infrastructure
│   ├── org-policies/
│   ├── vpc-host-prod/
│   └── logging/
├── production/
│   └── <service>/
│       └── terragrunt.hcl
├── non-production/
└── development/
```

### Terragrunt Configuration Pattern

```hcl
# environments/<env>/<component>/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

include "env" {
  path = "${get_terragrunt_dir()}/../../_env/<component>.hcl"
}

inputs = {
  environment = "production"
  # Environment-specific overrides
}
```
```

---

### 2.3 GCP Security Patterns (ADR-003, ADR-004, ADR-006)

**New Section for `references/security-compliance.md`:**

```markdown
## GCP Security Patterns (rs_infra)

### CMEK Encryption (ADR-003)

✅ DO: Use KMS Autokey for automatic encryption
```hcl
resource "google_kms_key_ring_import_job" "autokey" {
  provider = google-beta
  # Autokey configuration
}
```

❌ DON'T: Use Google-managed keys for sensitive data
```hcl
# Missing CMEK configuration
resource "google_storage_bucket" "data" {
  name = "my-bucket"
  # No encryption block = Google-managed keys
}
```

### Group-Based IAM (ADR-004)

✅ DO: Assign roles to groups, not individuals
```hcl
resource "google_project_iam_member" "admin" {
  project = var.project_id
  role    = "roles/editor"
  member  = "group:platform-admins@example.com"
}
```

❌ DON'T: Assign roles to individual users
```hcl
resource "google_project_iam_member" "admin" {
  project = var.project_id
  role    = "roles/editor"
  member  = "user:alice@example.com"  # Bad: individual user
}
```

### Organization Policies (ADR-006)

✅ DO: Enforce security at organization level
```hcl
resource "google_org_policy_policy" "disable_serial_port" {
  name   = "organizations/${var.org_id}/policies/compute.disableSerialPortAccess"
  parent = "organizations/${var.org_id}"

  spec {
    rules {
      enforce = "TRUE"
    }
  }
}
```

### Checkov Skip Pattern

When skipping security checks, document justification:
```hcl
#checkov:skip=CKV_GCP_32:Public access prevented by organization policy
resource "google_storage_bucket" "logs" {
  name = "audit-logs"
}
```
```

---

### 2.4 Agentic Workflow Integration (ADR-010)

**New Section for SKILL.md:**

```markdown
## Agentic Development Workflow

When working in an agentic workflow context, follow this 5-phase process:

### Phase 1: Planning
- Check existing ADRs before proposing changes
- Assess blast radius (which environments affected?)
- Generate implementation plan with tasks

### Phase 2: TDD (Test-Driven Development)
1. Generate unit test scaffolding FIRST
   ```bash
   # Create test file before implementation
   modules/<module>/tests/valid_inputs.tftest.hcl
   ```
2. Run tests (should fail)
   ```bash
   tofu test  # RED phase
   ```
3. Implement minimum code to pass
4. Run tests again (should pass)
   ```bash
   tofu test  # GREEN phase
   ```

### Phase 3: Implementation
- Follow existing patterns in codebase
- Run security scans continuously
- Keep changes focused and minimal

### Phase 4: Atomic Commits
- One logical change per commit
- Use conventional commit format:
  ```
  feat(vpc-host): add secondary IP range for GKE

  Refs: ADR-001
  ```

### Phase 5: Documentation Sync
- Run terraform-docs after module changes
- Update ADRs if architectural decisions changed
- Verify ARCHITECTURE.md accuracy
```

---

### 2.5 CI/CD Workflow Alignment

**Changes to `references/ci-cd-workflows.md`:**

```markdown
## rs_infra CI/CD Integration

### GitHub Actions Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `terragrunt-plan.yml` | PR | Lint, plan, security scan |
| `terragrunt-apply.yml` | Push to main | Apply changes |
| `integration-tests.yml` | PR/Schedule | Run Terratest suite |
| `terraform-docs.yml` | Module changes | Update READMEs |
| `drift-detection.yml` | Daily 6 AM | Detect configuration drift |

### Workload Identity Federation (ADR-008)

✅ DO: Use WIF for GitHub Actions authentication
```yaml
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
    service_account: ${{ secrets.WIF_SA }}
```

❌ DON'T: Use service account keys
```yaml
# Bad: JSON key in secrets
- uses: google-github-actions/auth@v2
  with:
    credentials_json: ${{ secrets.GCP_SA_KEY }}
```

### Pre-commit Hooks

Required hooks for local development:
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
  - repo: https://github.com/bridgecrewio/checkov
    hooks:
      - id: checkov
```
```

---

## 3. New Baseline Test Scenarios

Add these scenarios to `tests/baseline-scenarios.md`:

```markdown
## rs_infra-Specific Scenarios

### Scenario 9: TDD Workflow Adherence
**Prompt:** "Create a new module for Cloud SQL instances"
**Without Skill:** Agent implements module directly without tests
**With Skill:** Agent creates `.tftest.hcl` file first, runs RED phase, then implements

### Scenario 10: Terragrunt State Isolation
**Prompt:** "Add a new service project to production"
**Without Skill:** Agent may suggest monolithic state or incorrect path
**With Skill:** Agent creates proper `environments/production/<service>/terragrunt.hcl`

### Scenario 11: GCP Security Patterns
**Prompt:** "Create a storage bucket for audit logs"
**Without Skill:** Missing CMEK, missing access controls
**With Skill:** Includes CMEK via Autokey, uniform bucket-level access, logging

### Scenario 12: Atomic Commit Guidance
**Prompt:** "I've made several changes, help me commit"
**Without Skill:** Single large commit or unclear message
**With Skill:** Suggests splitting into atomic commits with conventional format

### Scenario 13: Documentation Sync
**Prompt:** "I added a new variable to the VPC module"
**Without Skill:** No reminder about documentation
**With Skill:** Prompts to run terraform-docs and check if ADRs need updates
```

---

## 4. Implementation Roadmap

### Phase 1: Core Updates (Week 1)

- [ ] Update `SKILL.md` with agentic workflow section
- [ ] Add rs_infra module structure to `references/module-patterns.md`
- [ ] Add GCP security patterns to `references/security-compliance.md`
- [ ] Update testing guidance in `references/testing-frameworks.md`

### Phase 2: Integration (Week 2)

- [ ] Add new baseline test scenarios (9-13)
- [ ] Update CI/CD patterns in `references/ci-cd-workflows.md`
- [ ] Add Terragrunt patterns for state management
- [ ] Test all scenarios with skill loaded

### Phase 3: Validation (Week 3)

- [ ] Run full TDD cycle (RED → GREEN → REFACTOR)
- [ ] Document any new rationalizations
- [ ] Update rationalization-table.md with counters
- [ ] Version bump and release

---

## 5. Compatibility Considerations

### Preserve Generic Value
The skill should remain useful for generic Terraform/OpenTofu projects while adding rs_infra-specific patterns. Use conditional sections:

```markdown
## Module Structure

### Standard Structure
[Generic module structure for any project]

### rs_infra Structure (GCP Foundation)
[Specific patterns for rs_infra repository]
```

### Token Budget
| Component | Current | After Changes | Budget |
|-----------|---------|---------------|--------|
| SKILL.md | 516 lines | ~600 lines | <650 |
| testing-frameworks.md | 563 lines | ~700 lines | <800 |
| module-patterns.md | 1,126 lines | ~1,300 lines | <1,500 |
| security-compliance.md | 470 lines | ~650 lines | <800 |
| ci-cd-workflows.md | 473 lines | ~600 lines | <700 |
| **Total** | ~4.1K lines | ~5K lines | <5.5K |

---

## 6. Success Criteria

### Quantitative
- [ ] All 13 baseline scenarios pass with skill
- [ ] Token budget maintained (<35K total)
- [ ] Validation workflow passes
- [ ] No regression in original 8 scenarios

### Qualitative
- [ ] Agent generates OpenTofu native tests first (TDD)
- [ ] Agent follows rs_infra module structure
- [ ] Agent includes GCP security patterns (CMEK, IAM groups)
- [ ] Agent suggests atomic commits with conventional format
- [ ] Agent prompts for documentation sync

---

## 7. Open Questions

1. **Scope:** Should this be a fork or a separate "gcp-foundation-skill"?
2. **Versioning:** Maintain sync with upstream or diverge?
3. **Distribution:** Private to RS org or publish to marketplace?
4. **Integration:** Should this skill auto-load with rs_infra CLAUDE.md?

---

## Appendix: File Change Summary

| File | Action | Description |
|------|--------|-------------|
| `SKILL.md` | Modify | Add agentic workflow section |
| `references/testing-frameworks.md` | Modify | Add OpenTofu native + Terratest patterns |
| `references/module-patterns.md` | Modify | Add rs_infra structure, Terragrunt |
| `references/security-compliance.md` | Modify | Add GCP patterns (CMEK, IAM, org policies) |
| `references/ci-cd-workflows.md` | Modify | Add rs_infra GitHub Actions, WIF |
| `tests/baseline-scenarios.md` | Modify | Add scenarios 9-13 |
| `tests/compliance-verification.md` | Modify | Add verification for new scenarios |
