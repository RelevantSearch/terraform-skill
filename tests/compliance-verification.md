# Compliance Verification (GREEN Phase)

> **Purpose:** Verify that terraform-skill changes agent behavior per TDD methodology
>
> **Prerequisite:** baseline-scenarios.md must be completed first (RED phase)

This document defines the GREEN phase of TDD testing: running the same scenarios WITH the skill loaded and verifying behavior changes.

---

## Testing Workflow

### Prerequisites

1. ✅ RED phase complete (`baseline-scenarios.md` scenarios run WITHOUT skill)
2. ✅ Baseline results documented in `baseline-results/` directory
3. ✅ Skill loaded in Claude environment

### GREEN Phase Process

For each scenario from `baseline-scenarios.md`:

1. **Load terraform-skill** in Claude environment
2. **Run exact same prompt** as baseline
3. **Document agent response** in `compliance-results/scenario-N.md`
4. **Compare to baseline** - what changed?
5. **Verify success criteria** from baseline scenario

---

## Comparison Template

For each scenario, document:

### Scenario N: [Name]

**Baseline Behavior (WITHOUT skill):**
- [What agent did/said]
- [What was missed]
- [Rationalizations used]

**Compliance Behavior (WITH skill):**
- [What agent did/said]
- [What improved]
- [Skill content referenced]

**Behavior Change:**
- ✅ **Improved:** [Specific improvements]
- ⚠️ **Partial:** [Partially addressed]
- ❌ **Unchanged:** [Still missing]

**Success Criteria Status:**
- [ ] Criterion 1
- [ ] Criterion 2
- [ ] etc.

**Evidence of Skill Usage:**
- [ ] Agent referenced decision matrix
- [ ] Agent quoted/paraphrased skill content
- [ ] Agent followed patterns from skill
- [ ] Agent used skill-specific terminology

**New Rationalizations Discovered:**
- [Any new excuses/workarounds to add to rationalization table]

---

## Scenario 1: Module Creation Without Testing

### Expected Improvements

**Baseline → Compliance Changes:**
- Agent NOW proactively mentions testing (not skips it)
- Agent uses testing decision matrix from SKILL.md:90-103
- Agent asks about Terraform version for framework selection
- Agent includes testing in deliverables OR asks user preference

### Success Criteria Verification

- [ ] Agent mentions testing proactively
- [ ] Agent uses testing decision matrix
- [ ] Agent asks about version for framework selection
- [ ] Agent doesn't rationalize skipping tests

### Evidence Checklist

Look for agent:
- Referencing "testing strategy framework"
- Mentioning "native tests (1.6+)" or "Terratest"
- Asking "What Terraform/OpenTofu version are you using?"
- Including test files in module structure

### Common Compliance Failures

If agent STILL skips testing:
- [ ] Check skill description triggers (may need enhancement)
- [ ] Check "When to Use This Skill" section clarity
- [ ] Add explicit counter-rationalization to SKILL.md

---

## Scenario 2: Choosing Testing Framework

### Expected Improvements

**Baseline → Compliance Changes:**
- Agent NOW asks clarifying questions (version, Go expertise, cost)
- Agent uses decision matrix instead of generic "use Terratest"
- Agent explains rationale for recommendation
- Agent considers multiple factors (not just defaults to one tool)

### Success Criteria Verification

- [ ] Agent asks version before recommending
- [ ] Agent uses decision matrix explicitly
- [ ] Agent explains rationale
- [ ] Agent considers cost implications
- [ ] Agent doesn't default to single recommendation

### Evidence Checklist

Look for agent:
- Directly referencing decision matrix table from SKILL.md
- Asking about "Go expertise on team"
- Mentioning "cost-sensitive workflow"
- Comparing multiple approaches

---

## Scenario 3: Security Scanning Omission

### Expected Improvements

**Baseline → Compliance Changes:**
- Agent NOW flags obvious security issues immediately
- Agent recommends trivy/checkov
- Agent references Security & Compliance section
- Agent provides specific fixes

### Success Criteria Verification

- [ ] Agent flags public S3 bucket
- [ ] Agent flags wide-open security group
- [ ] Agent recommends security scanning tools
- [ ] Agent provides secure alternatives
- [ ] Agent doesn't stop at "syntax correct"

### Evidence Checklist

Look for agent:
- Mentioning "trivy" or "checkov"
- Referencing security compliance guide
- Showing ✅ DO vs ❌ DON'T patterns
- Providing least-privilege examples

---

## Scenario 4: Naming Convention Violations

### Expected Improvements

**Baseline → Compliance Changes:**
- Agent NOW uses descriptive names (not generic)
- Agent follows naming conventions from SKILL.md:63-83
- Agent avoids anti-patterns without prompting

### Success Criteria Verification

- [ ] Resource names are descriptive and contextual
- [ ] Agent avoids generic names
- [ ] Variable names include context
- [ ] Follows naming section without prompting

### Evidence Checklist

Look for:
- `web_server` instead of `this`
- `application_logs` instead of `bucket`
- Context in variable names (`vpc_cidr_block` not `cidr`)

---

## Scenario 5: CI/CD Workflow Without Cost Optimization

### Expected Improvements

**Baseline → Compliance Changes:**
- Agent NOW includes cost optimization strategy
- Agent uses mocking for PRs, integration tests for main
- Agent includes cleanup and tagging
- Agent mentions cost proactively

### Success Criteria Verification

- [ ] Workflow uses cheap validation on PRs
- [ ] Expensive tests on main branch only
- [ ] Includes cleanup steps
- [ ] Tags test resources
- [ ] Agent mentions cost optimization proactively

### Evidence Checklist

Look for agent:
- Referencing cost optimization section from skill
- Mentioning "mock providers (1.7+)"
- Including auto-cleanup steps
- Suggesting Infracost integration

---

## Scenario 6: State File Management

### Expected Improvements

**Baseline → Compliance Changes:**
- Agent NOW includes encryption + security features
- Agent mentions state locking, access controls
- Agent provides concrete secure configuration
- Agent references security guide

### Success Criteria Verification

- [ ] Mentions encryption at rest
- [ ] Mentions encryption in transit
- [ ] Recommends state locking
- [ ] Suggests access controls/IAM
- [ ] Provides configuration example

---

## Scenario 7: Module Structure

### Expected Improvements

**Baseline → Compliance Changes:**
- Agent NOW provides complete structure with examples/ and tests/
- Agent explains purpose of each component
- Agent notes examples/ dual purpose (docs + fixtures)

### Success Criteria Verification

- [ ] Includes all standard files
- [ ] Mentions examples/ directory
- [ ] Mentions tests/ directory
- [ ] Explains versions.tf
- [ ] Notes examples as docs + fixtures

---

## Scenario 8: Variable Design Best Practices

### Expected Improvements

**Baseline → Compliance Changes:**
- Agent NOW includes descriptions, types, validation
- Agent marks sensitive variables correctly
- Agent adds validation blocks where appropriate
- Agent provides sensible defaults

### Success Criteria Verification

- [ ] All variables have descriptions
- [ ] Explicit type constraints
- [ ] Password marked sensitive
- [ ] Validation block for CIDR
- [ ] Sensible defaults

---

## Scenario 9: TDD Workflow Adherence

### Expected Improvements

**Baseline → Compliance Changes:**
- Agent NOW writes tests before implementation
- Agent follows RED-GREEN-REFACTOR cycle
- Agent references ADR-009 testing strategy
- Agent uses rs_infra test patterns

### Success Criteria Verification

- [ ] Agent creates test file first
- [ ] Agent runs tests to confirm RED phase
- [ ] Agent implements minimum code for GREEN
- [ ] Agent follows TDD workflow
- [ ] Agent references rs_infra patterns

### Evidence Checklist

Look for agent:
- Creating `.tftest.hcl` BEFORE `main.tf`
- Running `tofu test` to confirm failure
- Mentioning "RED phase" or "tests should fail"
- Using `mock_provider` pattern from skill
- Referencing ADR-009

---

## Scenario 10: Terragrunt State Isolation

### Expected Improvements

**Baseline → Compliance Changes:**
- Agent NOW uses proper environment structure
- Agent includes `_env/` pattern
- Agent maintains state isolation
- Agent references ADR-007

### Success Criteria Verification

- [ ] Correct directory structure
- [ ] Uses `_env/` include pattern
- [ ] Module reference correct
- [ ] State path isolated
- [ ] Follows ADR-007

### Evidence Checklist

Look for agent:
- Creating `environments/<env>/<workload>/terragrunt.hcl`
- Using `include "env"` block
- Referencing `${get_repo_root()}/modules//`
- Mentioning state isolation
- Referencing ADR-007

---

## Scenario 11: GCP Security Patterns

### Expected Improvements

**Baseline → Compliance Changes:**
- Agent NOW includes CMEK encryption
- Agent uses group-based IAM
- Agent follows ADR-003 and ADR-004
- Agent produces Checkov-compliant code

### Success Criteria Verification

- [ ] CMEK encryption included
- [ ] KMS key referenced
- [ ] Group-based IAM used
- [ ] Follows ADR patterns
- [ ] Checkov compliant

### Evidence Checklist

Look for agent:
- Adding `encryption { default_kms_key_name = ... }` block
- Using `member = "group:..."` for IAM
- Mentioning ADR-003 (CMEK) or ADR-004 (IAM)
- Not using `user:xxx@` bindings
- Referencing security compliance guide

---

## Scenario 12: Atomic Commit Guidance

### Expected Improvements

**Baseline → Compliance Changes:**
- Agent NOW suggests splitting unrelated changes
- Agent uses conventional commit format
- Agent includes proper scope
- Agent mentions pre-commit validation

### Success Criteria Verification

- [ ] Suggests splitting changes
- [ ] Conventional format used
- [ ] Correct type identified
- [ ] Scope (module name) included
- [ ] Pre-commit mentioned

### Evidence Checklist

Look for agent:
- Identifying multiple logical changes
- Suggesting separate commits
- Using format: `<type>(<scope>): <description>`
- Recommending `pre-commit run`
- Referencing atomic commit skill

---

## Scenario 13: Documentation Sync

### Expected Improvements

**Baseline → Compliance Changes:**
- Agent NOW reminds about terraform-docs
- Agent considers ARCHITECTURE.md
- Agent checks ADR alignment
- Agent suggests doc-sync workflow

### Success Criteria Verification

- [ ] Mentions terraform-docs
- [ ] Considers ARCHITECTURE.md
- [ ] References ADRs
- [ ] Suggests doc-sync
- [ ] Provides specific command

### Evidence Checklist

Look for agent:
- Providing `terraform-docs markdown table` command
- Asking if topology changed (ARCHITECTURE.md)
- Checking ADR alignment
- Mentioning `/doc-sync` command
- Referencing doc sync skill

---

## Overall Compliance Assessment

### Passing Criteria

Skill is considered "passing GREEN phase" when:

**Quantitative:**
- [ ] 13/13 scenarios show measurable behavior improvement
- [ ] 80%+ of success criteria met across all scenarios
- [ ] Agent references skill content in 11/13+ scenarios

**Qualitative:**
- [ ] Agent proactively applies patterns (not reactive)
- [ ] Agent uses decision frameworks unprompted
- [ ] Agent cites specific sections/examples from skill
- [ ] Responses align with skill philosophy

### Failure Modes

If scenarios fail (no behavior change):

**Diagnosis:**
1. Check skill description - does it match trigger conditions?
2. Check "When to Use" section - clear enough?
3. Check content organization - is pattern findable?
4. Check keyword coverage - would search find it?

**Remediation:**
1. Enhance CSO (description, keywords)
2. Reorganize content for scannability
3. Add explicit counter-rationalizations
4. Re-test in REFACTOR phase

---

## Documentation Requirements

### For Each Scenario

Create file: `compliance-results/scenario-N-[name].md`

**Required sections:**
1. Full agent response (verbatim or screenshot)
2. Comparison to baseline (what changed)
3. Success criteria checklist
4. Evidence of skill usage
5. New rationalizations discovered
6. PASS/PARTIAL/FAIL verdict

### Summary Report

Create file: `compliance-results/SUMMARY.md`

**Include:**
- Overview: N/8 scenarios passed
- Success criteria: N% met overall
- Key improvements observed
- Remaining gaps
- Rationalizations to address in REFACTOR phase

---

## GREEN Phase Complete When:

- [ ] All 13 scenarios run WITH skill loaded
- [ ] Results documented in `compliance-results/` directory
- [ ] Comparison to baseline complete for all scenarios
- [ ] Success criteria evaluated
- [ ] Summary report written
- [ ] New rationalizations captured for REFACTOR phase

---

## Next Steps

After GREEN phase:
1. → `rationalization-table.md` - Update with findings
2. → REFACTOR phase - Add counters to SKILL.md for new rationalizations
3. → Re-test scenarios that failed or partially passed
4. → Iterate until 8/8 scenarios pass

**This is iterative:** First pass may only get 5/8 scenarios passing. That's expected. The goal is continuous improvement through the RED-GREEN-REFACTOR cycle.
