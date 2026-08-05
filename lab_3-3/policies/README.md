# Policies

OPA/Rego policies enforced against Terraform `planned_values` for this lab. Each policy maps to a NIST 800-53 control and runs against the plan before apply.

---

## AC-3 — Access Enforcement

**File:** [`ac3_no_public.rego`](ac3_no_public.rego)
**Severity:** Critical

Blocks public exposure at two layers:
- GCS buckets must set `uniform_bucket_level_access = true` and `public_access_prevention = "enforced"`.
- Firewall rules must not open management ports (22, 3389) to `0.0.0.0/0` or `*`.

**Remediation:** Set `uniform_bucket_level_access = true` and `public_access_prevention = "enforced"` on the bucket. For firewalls, narrow `source_ranges` or remove the rule entirely.

---

## SC-28 — Encryption at Rest

**File:** [`sc28_encryption.rego`](sc28_encryption.rego)
**Severity:** High

Every `google_storage_bucket` must encrypt at rest using a customer-managed encryption key (CMEK) rather than the Google-managed default.

**Remediation:** Add an `encryption { default_kms_key_name = ... }` block referencing a `google_kms_crypto_key` you control.

---

## CM-6 — Configuration Settings

**File:** [`cm6_required_tags.rego`](cm6_required_tags.rego)
**Severity:** Medium

Every taggable resource (`google_storage_bucket`, `google_compute_instance`, `google_compute_disk`) must carry all four required labels: `project`, `environment`, `managed_by`, `compliance_scope`.

**Remediation:** Add a `labels` block to the resource containing all four required keys, e.g. `labels = { project = "...", environment = "...", managed_by = "...", compliance_scope = "..." }`.

---

Tests for each policy live under [`tests/`](tests/).