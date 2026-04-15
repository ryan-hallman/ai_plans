# Terraform Linting Design

**Date**: 2026-04-13
**Status**: Approved design for implementation

---

## 1. Goal

Add repo-standard Terraform linting and security scanning that runs:

- locally through `pre-commit`
- explicitly in CI
- on demand through `Makefile` targets

The design should cover both best-practice/static analysis and security scanning for the Terraform code under `infrastructure/terraform/`.

---

## 2. Tooling Choice

### Best-practice linting: `tflint`

Use `tflint` as the primary Terraform linter. It is the standard choice for:

- Terraform language checks
- AWS provider best-practice rules
- invalid or suspicious Terraform patterns

The repo should include a committed `.tflint.hcl` so rule behavior is explicit and versioned.

### Security scanning: `trivy config`

Use `trivy config` for Terraform security scanning instead of adding `tfsec`.

Reason:

- it covers Terraform security scanning well
- it is actively maintained
- it avoids adding a second overlapping security scanner

This design does not include `checkov`, `terrascan`, or `tfsec` unless a later need appears.

---

## 3. Execution Model

### Pre-commit

Add Terraform hooks to `.pre-commit-config.yaml` so engineers get local feedback before commit.

Scope them to:

- `infrastructure/terraform/**`

The pre-commit layer should run:

- `tflint`
- `trivy config`

### CI

Add explicit Terraform lint/security steps to `.github/workflows/terraform-ci.yml`.

CI should not rely only on running the pre-commit hooks. The Terraform checks should appear as dedicated steps so failures are readable and operationally obvious.

CI should run:

- `tflint`
- `trivy config`

alongside the existing Terraform fmt/validate/local-plan flow.

### Makefile

Expose direct commands for local developer use:

- `make tf-lint`
- `make tf-sec`
- `make tf-check`

Where:

- `tf-lint` runs `tflint`
- `tf-sec` runs `trivy config`
- `tf-check` runs both in sequence

---

## 4. Repository Files

Implementation should touch:

- `.pre-commit-config.yaml`
- `.github/workflows/terraform-ci.yml`
- `Makefile`
- `.tflint.hcl`

### `.tflint.hcl`

This file should:

- enable the core Terraform ruleset
- enable the AWS ruleset
- pin plugin source/version explicitly

Keep the config minimal. Do not turn this into a large rule-suppression file without evidence.

---

## 5. CI Behavior

The Terraform CI workflow should continue to run:

- `terraform fmt -check`
- `make tf-validate`
- LocalStack-backed local plan

and add dedicated steps for:

- installing or setting up `tflint`
- running `tflint --init`
- running `tflint --recursive`
- installing `trivy` or using the official action
- running `trivy config infrastructure/terraform`

The Terraform lint/security steps should fail the workflow on findings unless explicitly configured otherwise.

---

## 6. Scope Boundaries

This design covers only Terraform linting/security for the current repo.

Out of scope:

- auto-fixing Terraform findings
- policy-as-code platforms
- mandatory severity thresholds beyond sane defaults
- AWS-authenticated security validation
- replacing current Python/web linting

---

## 7. Success Criteria

The change is complete when:

- Terraform lint/security tools are committed to repo config
- `pre-commit` runs them for Terraform files
- CI runs them explicitly
- developers can run the same checks with Makefile targets
- the branch validates cleanly after the tooling is added
