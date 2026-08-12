# cgep

Repository for Certified GRC Engineer Labs — hands-on labs that implement GRC (Governance, Risk, and Compliance) controls as code: Terraform infrastructure mapped to NIST 800-53 controls, OPA/Rego policy gates, CI enforcement, and signed, tamper-evident evidence pipelines.

## Repository structure

Each `lab_*` directory is a self-contained lab covering a stage of the "compliance as code" pipeline:

| Lab | Focus |
|---|---|
| [lab_2-3](lab_2-3/) | Compliant AWS S3 bucket primitive (SC-28, AU-3, AU-6, CM-6, AC-3) built in Terraform, with plan/state evidence capture. |
| [lab_2-4](lab_2-4/) | Compliant GCS bucket module (SC-12, SC-13, SC-28, AU-11, CM-6) with dev/prod/negative-test consumers and a compliance attestation. |
| [lab_2-5](lab_2-5/) | Evidence capture scripting — bundles Terraform plan/state output and uploads it to an evidence store. |
| [lab_3-3](lab_3-3/) | OPA/Rego policies (AC-3, SC-28, CM-6) enforced against Terraform plan JSON, with unit tests and evidence output. |
| [lab_3-4](lab_3-4/) | Multi-cloud (AWS + GCP) version of the policy gate, driven by [`scripts/policy-gate.sh`](lab_3-4/scripts/policy-gate.sh) and Conftest. |
| [lab_4-3](lab_4-3/) | Full GRC evidence pipeline: GitHub Actions OIDC to AWS, Conftest + tfsec gating, Cosign keyless signing, and retention-locked evidence storage. See [WRITEUP.md](lab_4-3/WRITEUP.md) for the chain-of-custody design. |

Each lab typically contains:
- `terraform/` — infrastructure definitions for the control(s) under test
- `policies/` — OPA/Rego policies mapped to NIST 800-53 controls, with a `tests/` subdirectory
- `scripts/` — gate/evidence automation (e.g. `policy-gate.sh`, `capture-evidence.sh`)
- `evidence/` — captured plan/state/policy-test output demonstrating control enforcement

The CI workflow at [.github/workflows/grc-gate.yml](.github/workflows/grc-gate.yml) runs the full gate (Terraform plan → Conftest → tfsec → Cosign sign → evidence vault upload) on pull requests into `main`.

## Setup

### Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform) 1.6+
- [Conftest](https://www.conftest.dev/) 0.50+
- [tfsec](https://github.com/aquasecurity/tfsec)
- [Cosign](https://github.com/sigstore/cosign) 2.2+ (only needed for the lab_4-3 signing pipeline)
- Python 3 (used by gate scripts to parse JSON output)
- AWS CLI and/or `gcloud` CLI, configured with credentials for whichever lab you're running (labs target AWS, GCP, or both — see the table above)

### Running a lab locally

From within a lab's Terraform directory:

```bash
terraform init
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
```

Then run the policy gate against the plan, e.g. for lab_3-4 or lab_4-3:

```bash
./scripts/policy-gate.sh --workspace <path-to-terraform-dir> --policy policies
```

Or run Conftest directly against a specific policy namespace:

```bash
conftest test --policy policies --namespace compliance.ac3_aws plan.json
```

Policy unit tests (where present) run via:

```bash
opa test policies/tests
```

### CI / evidence pipeline (lab_4-3)

The GitHub Actions workflow requires repository variables `AWS_ROLE_ARN` (OIDC role) and `EVIDENCE_VAULT` (S3 bucket name) to be configured, plus `id-token: write` permission (already set in the workflow) for keyless OIDC/Cosign signing. It runs automatically on PRs to `main`, or manually via `workflow_dispatch`.

### Notes

- `.terraform/`, `.terraform.lock.hcl`, and `*.tfstate*` are gitignored — run `terraform init` locally before planning.
- `evidence/` directories contain committed example output (plan JSON, test results, attestations) for reference; regenerate them by re-running the relevant lab's plan/gate steps.
