# Capstone Writeup
## Acme Health Patient Intake API — GRC Baseline

---

## 1. Framework Selection: Why SOC 2 Type II

The three candidate frameworks were HIPAA Security Rule, SOC 2 Trust Services Criteria (TSC), and CMMC Level 2.

**I selected SOC 2 Type II as the primary framework** for the following reasons:

- **Business driver**: The immediate driver is enterprise customer trust. A customer has asked for a SOC 2 report before signing. HIPAA is a legal floor, not a competitive differentiator at this stage. SOC 2 is what closes deals.
- **Scope fit**: SOC 2 TSC (CC6.x, CC7.x, A1.x) maps directly to the gaps in the starter workload — encryption at rest (CC6.1), transmission security (CC6.7), boundary protection (CC6.6), least privilege (CC6.3), and monitoring (CC7.2). Every gap had a clear TSC home.
- **CMMC exclusion**: CMMC Level 2 applies to DoD contractors handling CUI. Acme Health has no current federal contracts. Implementing CMMC would add compliance overhead with no near-term return.
- **HIPAA cross-reference**: HIPAA Technical Safeguard controls (164.312) are satisfied by the same remediations that close SOC 2 gaps. This is documented in `GAPS.md`. HIPAA is addressed as a byproduct, not a separate workstream.

**NIST SP 800-53 Rev 5** is used as the underlying control catalog in OSCAL because it has a published machine-readable catalog and SOC 2 TSC maps onto it. The OSCAL component definition references NIST control IDs with SOC 2 TSC noted in `props`.

---

## 2. Gap Remediation Summary

All 8 gaps from `GAPS.md` were addressed. The table below summarizes each gap, the remediation approach, and the SOC 2 TSC control closed.

| Gap | Description | Remediation | SOC 2 TSC | Status |
|---|---|---|---|---|
| GAP-01 | S3 uploads bucket uses AWS-managed SSE-S3, not CMK | `module.baseline.aws_kms_key.phi` + `aws_s3_bucket_server_side_encryption_configuration.uploads` with `sse_algorithm = "aws:kms"` | CC6.1 | Closed |
| GAP-02 | DynamoDB table uses AWS-owned default key | New `aws_dynamodb_table.intake_cmk` with `server_side_encryption { kms_key_arn }`. Existing table flagged by Rego policy. | CC6.1 | Closed (new table) |
| GAP-03 | S3 uploads bucket missing TLS-only bucket policy | `module.baseline.aws_s3_bucket_policy.uploads` with `DenyNonTLS` statement on `aws:SecureTransport=false` | CC6.7 | Closed |
| GAP-04 | S3 uploads bucket has no versioning | `module.baseline.aws_s3_bucket_versioning.uploads` with `status = "Enabled"` | A1.2 | Closed |
| GAP-05 | Lambda runs outside VPC | `vpc_config` block added to `aws_lambda_function.intake` with private subnets + `module.baseline.aws_security_group.lambda` (HTTPS egress only). VPC Gateway endpoints for DynamoDB and S3. | CC6.6 | Closed |
| GAP-06 | Lambda has no DLQ or observability | `module.baseline.aws_sqs_queue.lambda_dlq` + `aws_lambda_function_event_invoke_config.intake` routing failures to DLQ. `tracing_config.mode = "Active"` for X-Ray. | CC7.2 | Closed |
| GAP-07 | Lambda IAM role grants `dynamodb:*` and `s3:*` | `aws_iam_role_policy.lambda_least_privilege` replaces wildcards with `dynamodb:PutItem/GetItem/UpdateItem/Query` and `s3:PutObject/GetObject`. Rego policy `gap07_iam_least_privilege.rego` detects regressions. | CC6.3 | Closed |
| GAP-08 | API Gateway has no access logging or throttling | `aws_apigatewayv2_stage.default_hardened` with `access_log_settings` to CloudWatch log group `/aws/apigateway/acme-health-intake-316391d2`. Throttle: 100 burst / 50 rate. CloudTrail provides AWS API-level audit trail. | CC7.2 | Closed |

---

## 3. Design Decisions and Trade-offs

### Object Lock Mode: GOVERNANCE vs COMPLIANCE

The evidence vault uses `GOVERNANCE` mode with 90-day retention. **COMPLIANCE mode** would be stronger — it cannot be overridden by anyone, including account root, and requires AWS Support to remove. However:

- This is a lab environment. COMPLIANCE mode would prevent cleanup at the end of the lab period without opening an AWS Support ticket.
- In a production engagement, COMPLIANCE mode is the correct choice for evidence that must be tamper-proof for audit periods (typically 1 year for SOC 2).
- GOVERNANCE mode still requires a specific IAM permission (`s3:BypassGovernanceRetention`) to override, protecting against accidental or unauthorized deletion by operators without that permission.

**Production recommendation**: Switch to `COMPLIANCE` mode with a 365-day retention period before the first SOC 2 audit window opens.

### Terraform State: Remote S3 Backend

Terraform state is stored in `acme-health-intake-tfstate-316391d2` with versioning enabled. This was required for the CI/CD pipeline to share state with local development runs. State is not encrypted with a CMK in this implementation — a production hardening would add KMS encryption on the state bucket and DynamoDB locking.

### GAP-02 Design: New Table vs In-Place Modification

DynamoDB does not support changing the encryption key on an existing table. The correct production approach is a blue/green migration: create a new table with the CMK, backfill data, cut over the Lambda environment variable, and decommission the old table. For the capstone, a new `intake_cmk` table is provisioned with the CMK. The Rego policy `gap02_dynamodb_cmk.rego` enforces that any newly created table must have CMK encryption, which closes the gap for future deployments.

### Pipeline: No Manual Approval Gate

The pipeline applies automatically on merge to `main`. A manual gate between plan and apply would satisfy a Change Advisory Board (CAB) process common in regulated enterprises. This was omitted because:

1. The policy gate (Conftest) is the primary control — it runs pre-merge on every PR.
2. Adding a manual gate without also enforcing branch protection and required reviews creates a false sense of control.

**Production recommendation**: Enable required reviewers on `main`, enforce the policy gate as a required status check, and add a manual approval job between plan and apply for production environment changes.

### github_actions IAM Role: Broad Permissions

The `aws_iam_role_policy.github_actions` policy uses broad permissions (`ec2:*`, `lambda:*`, `iam:*`) to allow the pipeline to run full Terraform applies. This policy itself triggers the `gap07` Rego rule. The policy is scoped to the pipeline role only, not the Lambda execution role.

**This is a known and accepted trade-off**: CI/CD pipelines that manage infrastructure require broad permissions to function. The mitigation is the OIDC trust condition (`StringLike: repo:davin1121/cgep-capstone:*`) which restricts assumption to only this specific repository. In a production environment, permissions would be further scoped to specific resource ARNs using Terraform `for_each` outputs.

---

## 4. Known Remaining Gaps

| Item | Risk | Recommended Remediation |
|---|---|---|
| `aws_dynamodb_table.intake` uses AWS-owned key | Medium — existing data not under CMK | Blue/green table migration with Lambda env var cutover |
| `github_actions` role uses broad IAM permissions | Low — scoped to this repo via OIDC | Scope to specific resource ARNs post-capstone |
| Evidence vault uses GOVERNANCE not COMPLIANCE Object Lock | Low — lab environment | Switch to COMPLIANCE mode before first audit window |
| API Gateway WAF not implemented | Medium — no L7 protection | Add `aws_wafv2_web_acl` association in a follow-on sprint |
| No patient data lifecycle policy | Medium — HIPAA right-of-access, GDPR erasure | Implement DynamoDB TTL and S3 lifecycle rules |
| Terraform state bucket unencrypted with CMK | Low | Add KMS encryption + DynamoDB state locking |

---

## 5. Evidence Chain of Custody

Every merge to `main` triggers the `grc-gate` pipeline which:

1. Runs `terraform plan` and exports `tfplan.json`
2. Runs `opa test` (13/13 must pass) and `opa eval` (deny count must be 0)
3. Runs `terraform apply`
4. Bundles `tfplan.json` + `policy-results.txt` into a timestamped tarball
5. Signs the bundle with Cosign keyless signing (identity pinned to the GitHub Actions OIDC token)
6. Uploads the bundle, SHA-256 digest, Cosign signature bundle, and `receipt.json` to the evidence vault `acme-health-intake-evidence-vault-316391d2` under `runs/<run_id>/`

The vault bucket has Object Lock (GOVERNANCE, 90 days) preventing deletion. The `receipt.json` records run ID, commit SHA, actor, and timestamp. An auditor can:

1. List `s3://acme-health-intake-evidence-vault-316391d2/runs/` to enumerate all pipeline runs
2. Download a bundle and its `.sig.bundle`
3. Verify with `cosign verify-blob --bundle <sig.bundle> <bundle.tar.gz>` against the Sigstore Rekor transparency log
4. Confirm the `tfplan.json` inside shows the expected control configurations

This constitutes a cryptographically verifiable chain of custody from code commit to infrastructure state.

---

## 6. OSCAL Documentation

`oscal/component-definition.json` documents seven implemented requirements mapping NIST 800-53 Rev 5 controls to the workload:

| NIST Control | SOC 2 TSC | Gaps Closed |
|---|---|---|
| SC-28 (Protection of Information at Rest) | CC6.1 | GAP-01, GAP-02 |
| SC-7 (Boundary Protection) | CC6.6 | GAP-05 |
| SI-7 (Software Integrity / Monitoring) | CC7.2 | GAP-06, GAP-08 |
| AC-6 (Least Privilege) | CC6.3 | GAP-07 |
| AU-2 (Audit Record Generation) | CC7.2 | GAP-08 (partial) |
| A-1.2 (Availability / Recovery) | A1.2 | GAP-04 |
| CC-6.7 (Transmission Security) | CC6.7 | GAP-03 |

`oscal/profile.json` selects the corresponding NIST 800-53 Rev 5 control IDs from the NIST published catalog.
