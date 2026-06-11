# cgep-capstone — Acme Health Patient Intake API GRC Baseline

> SOC 2 Type II GRC baseline for the Acme Health Patient Intake API. Built on the `cgep-app-starter` workload.

[![GRC Gate](https://github.com/davin1121/cgep-capstone/actions/workflows/grc-gate.yml/badge.svg)](https://github.com/davin1121/cgep-capstone/actions/workflows/grc-gate.yml)

## What this is

This repository wraps the deliberately non-compliant `cgep-app-starter` workload with four CGE-P capstone layers:

| Layer | What | Where |
|---|---|---|
| **1 — Terraform baseline** | KMS CMK, S3 evidence vault (Object Lock), CloudTrail, Lambda VPC + SG + DLQ, API GW logging, least-privilege IAM | `terraform/baseline/` |
| **2 — Rego policy suite** | 5 OPA policies covering GAP-01/02/03/05/07, 13 unit tests | `policies/` |
| **3 — GitHub Actions pipeline** | Plan → Policy gate → Apply → Cosign sign → vault upload | `.github/workflows/grc-gate.yml` |
| **4 — OSCAL component** | Component definition + profile (NIST 800-53 Rev 5 mapped to SOC 2 TSC) | `oscal/` |

**Primary framework:** SOC 2 Type II (Trust Services Criteria)
**NIST catalog:** SP 800-53 Rev 5 (used as OSCAL catalog source)

---

## Gaps closed

All 8 gaps from `GAPS.md` are remediated. See `WRITEUP.md` for full details.

| Gap | SOC 2 TSC | Remediation |
|---|---|---|
| GAP-01: S3 no CMK | CC6.1 | SSE-KMS with `alias/acme-health-intake-phi` |
| GAP-02: DynamoDB no CMK | CC6.1 | New `intake_cmk` table with CMK + Rego policy |
| GAP-03: S3 no TLS deny | CC6.7 | Bucket policy denying `aws:SecureTransport=false` |
| GAP-04: S3 no versioning | A1.2 | Versioning enabled on uploads bucket |
| GAP-05: Lambda not in VPC | CC6.6 | `vpc_config` + SG + VPC Gateway endpoints |
| GAP-06: Lambda no DLQ | CC7.2 | SQS DLQ + X-Ray active tracing |
| GAP-07: IAM wildcard actions | CC6.3 | Least-privilege policy + Rego enforcement |
| GAP-08: No API GW logging | CC7.2 | CloudWatch log group + access log format |

---

## Repository layout

```
cgep-capstone/
├── README.md
├── WRITEUP.md               # Capstone writeup: framework choice, decisions, trade-offs
├── DESIGN.md                # Architecture design document
├── GAPS.md                  # Original gap definitions (from starter)
├── Makefile
├── terraform/
│   ├── main.tf              # Root module: all gap remediations + OIDC role
│   ├── github_oidc.tf       # GitHub Actions OIDC role
│   ├── variables.tf
│   ├── outputs.tf
│   ├── baseline/            # GRC baseline module
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── lambda/handler.py
├── policies/
│   ├── gap01_s3_cmk.rego
│   ├── gap02_dynamodb_cmk.rego
│   ├── gap03_s3_tls.rego
│   ├── gap05_lambda_vpc.rego
│   ├── gap07_iam_least_privilege.rego
│   └── tests/
├── oscal/
│   ├── component-definition.json
│   └── profile.json
└── .github/workflows/
    └── grc-gate.yml
```

---

## Deploy

```powershell
aws sts get-caller-identity   # confirm correct account
cd terraform
terraform init
terraform apply -auto-approve
```

## Run policy tests locally

```powershell
opa test ./policies -v
```

## Smoke test

```powershell
[System.IO.File]::WriteAllText("body.json", '{"patient_id":"P-0001","fields":{"reason":"smoke-test"}}')
curl.exe -sS -X POST "https://3hzxmmnws7.execute-api.us-east-1.amazonaws.com/intake" -H "content-type: application/json" --data-binary "@body.json"
```

---

## What this is

A minimal AWS workload: VPC, Lambda, API Gateway, DynamoDB, S3. It ingests patient intake submissions over HTTPS. Think of it as a system you have just inherited from an engineering team and been asked to make audit-defensible.

This repository ships **non-compliant on purpose**. Your job in the capstone is not to rewrite this app. Your job is to wrap it with the four CGE-P layers (Terraform GRC baseline, Rego policies, GitHub Actions evidence pipeline, OSCAL component) so the same workload becomes audit-defensible against HIPAA, SOC 2, and CMMC L2.

## The deploy gate

If you cannot deploy this starter, you cannot pass the capstone. Real GRC engineers inherit working systems. Step zero is making the system run.

```bash
git clone https://github.com/GRCEngClub/cgep-app-starter
cd cgep-app-starter

# Confirm you're authenticated to the right account:
make creds AWS_PROFILE=<your-sandbox-profile>

make deploy AWS_PROFILE=<your-sandbox-profile>
make test    AWS_PROFILE=<your-sandbox-profile>
```

> **AWS SSO note:** if your profile is SSO-based, Terraform's AWS provider can fail to read it directly with `failed to find SSO session section`. The Makefile's `eval $(aws configure export-credentials)` pattern handles this. If you're running `terraform` commands by hand, do the same export first.

Expected output of `make test`:

```json
{
    "submission_id": "f1e3...",
    "status": "received"
}
```

When you're done exploring: `make destroy`.

## What you build on top

Fork the repo into your own `cgep-capstone` and add:

1. **Layer 1 — GRC baseline (Terraform).** KMS keys, an S3 evidence vault with Object Lock, a CloudTrail trail. Bring this starter's data stores under your CMK.
2. **Layer 2 — OPA policy suite (Rego).** Five or more policies that catch the named gaps in [GAPS.md](GAPS.md). Each policy maps to at least one control from the framework you choose.
3. **Layer 3 — GitHub Actions pipeline.** Plan → Conftest gate → apply → Cosign sign → upload to vault.
4. **Layer 4 — OSCAL component.** A `component-definition.json` describing how your governed system implements its controls.

Full brief: `docs/labs/07_01_capstone_brief.md` in the course content repo.

## Framework mapping is required

Your capstone must declare a primary framework: **HIPAA Security Rule**, **SOC 2 Trust Services Criteria**, or **CMMC Level 2**. Every policy carries at least one control ID from your chosen framework. Your OSCAL component's `control-implementations` reference your framework's catalog.

A starter mapping is in [FRAMEWORKS.md](FRAMEWORKS.md). It is not the only valid mapping. You're expected to defend yours.

## Cost

Roughly $0 if destroyed within an hour. Lambda + API Gateway + DynamoDB + S3 are all pay-per-use, and an empty deployment generates no traffic. CloudTrail (which you add) costs cents.

## Layout

```
cgep-app-starter/
├── README.md            # this file
├── WORKLOAD.md          # what the API does
├── GAPS.md              # the named flaws your policies must catch
├── FRAMEWORKS.md        # HIPAA / SOC 2 / CMMC mapping primer
├── Makefile             # make deploy | test | destroy
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── lambda/handler.py
└── test/
    └── intake.sh
```

## License

MIT. Fork freely. Submissions remain learners' own work.
