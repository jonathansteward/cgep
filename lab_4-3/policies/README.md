# Policies

OPA/Rego policies enforced against Terraform `planned_values` for this lab. Each policy maps to a NIST 800-53 control and runs against the plan before apply. Every control has a GCP version and an AWS version.

---

## AC-3 — Access Enforcement

**Severity:** Critical

- [`ac3_no_public.rego`](ac3_no_public.rego) — **GCP.** Blocks public exposure at two layers: GCS buckets must set `uniform_bucket_level_access = true` and `public_access_prevention = "enforced"`; firewall rules must not open management ports (22, 3389) to `0.0.0.0/0` or `*`.
- [`ac3_no_public_aws.rego`](ac3_no_public_aws.rego) — **AWS.** Every `aws_s3_bucket` must have a matching `aws_s3_bucket_public_access_block` with all four block-public flags set to `true`.

**Remediation:** GCP — set `uniform_bucket_level_access = true` and `public_access_prevention = "enforced"` on the bucket, and narrow or remove risky firewall rules. AWS — add an `aws_s3_bucket_public_access_block` for the bucket with all four flags `true`.

---

## SC-28 — Encryption at Rest

**Severity:** High

- [`sc28_encryption.rego`](sc28_encryption.rego) — **GCP.** Every `google_storage_bucket` must encrypt at rest using a customer-managed encryption key (CMEK) rather than the Google-managed default.
- [`sc28_encryption_aws.rego`](sc28_encryption_aws.rego) — **AWS.** Every `aws_s3_bucket` must have a matching `aws_s3_bucket_server_side_encryption_configuration`.

**Remediation:** GCP — add an `encryption { default_kms_key_name = ... }` block referencing a `google_kms_crypto_key` you control. AWS — add an `aws_s3_bucket_server_side_encryption_configuration` referencing the bucket.

---

## CM-6 — Configuration Settings

**Severity:** Medium

- [`cm6_required_tags.rego`](cm6_required_tags.rego) — **GCP.** Every `google_storage_bucket`, `google_compute_instance`, and `google_compute_disk` must carry all four required labels.
- [`cm6_required_tags_aws.rego`](cm6_required_tags_aws.rego) — **AWS.** Every `aws_s3_bucket`, `aws_dynamodb_table`, `aws_lambda_function`, `aws_kms_key`, and `aws_cloudtrail` must carry all four required tags.

The four required keys are the same on both clouds: `project`, `environment`, `managed_by`, `compliance_scope`.

**Remediation:** Add a `labels` block (GCP) or `tags` block (AWS) containing all four keys, e.g. `{ project = "...", environment = "...", managed_by = "...", compliance_scope = "..." }`.

---

Tests for each policy live under [`tests/`](tests/).
