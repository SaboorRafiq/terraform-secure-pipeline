# Terraform Secure Pipeline

A Terraform IaC pipeline with automated security scanning (Checkov) enforced via GitHub Actions before merge. Built to demonstrate the actual security engineering loop — insecure baseline, scan, remediate, enforce — not just a syntax exercise.

## What this is

A minimal AWS S3 bucket deployment, deliberately misconfigured at first, then remediated against Checkov's policy checks, with a CI pipeline that blocks future regressions automatically.

## Architecture

`main.tf` — S3 bucket + public access block, versioning, KMS encryption, access logging
`.github/workflows/terraform-security.yml` — runs on every push/PR to `main`:
  1. `terraform fmt -check`
  2. `terraform init`
  3. `terraform validate`
  4. Checkov scan (blocking — pipeline fails if new findings appear)

## Remediation history

Initial commit shipped an intentionally insecure bucket: public access enabled, no encryption, no versioning, no logging. Checkov flagged 11 failed checks (5 passed / 11 Failed).

Fixed:
- Enabled all four S3 public access block settings (CKV_AWS_53, CKV_AWS_54, CKV_AWS_55, CKV_AWS_56, CKV2_AWS_6)
- Enabled versioning (CKV_AWS_21)
- Enabled KMS encryption at rest (CKV_AWS_145)
- Enabled access logging (CKV_AWS_18)

Result: 13 passed / 3 failed.

**Deliberately deferred, with inline `checkov:skip` annotations and reasoning:**
- CKV_AWS_144 (cross-region replication) — requires a second bucket + IAM role, out of scope for this demo
- CKV2_AWS_61 (lifecycle configuration) — not required for demo data
- CKV2_AWS_62 (event notifications) — requires SNS/Lambda wiring, out of scope

These aren't gaps that were missed — they're documented risk-acceptance decisions, same pattern used when triaging findings against acceptable risk in a production vulnerability management program.

Full diff: [baseline commit](https://github.com/SaboorRafiq/terraform-secure-pipeline/commit/a4e2fff) → [remediation commit](https://github.com/SaboorRafiq/terraform-secure-pipeline/commit/ed511d2)

## CI enforcement

GitHub Actions also scans its own workflow file — CKV2_GHA_1 flagged the pipeline's default `write-all` permissions, fixed by scoping to `contents: read` explicitly. See [Actions runs](https://github.com/SaboorRafiq/terraform-secure-pipeline/actions) for full scan history on every commit.

## Why this exists

Built to get hands-on reps with the exact pattern I work with professionally: IaC security scanning, policy-as-code gating, CI/CD pipeline hardening. Not a tutorial copy — every finding here was actually hit, read, and fixed, not pre-solved.

This mirrors work I do at Bentley evaluating and remediating CI/CD pipeline risk — specifically, treating pipeline configuration as untrusted-by-default and gating it against automated policy checks before it runs, rather than trusting it after the fact.
