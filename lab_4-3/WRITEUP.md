# Lab 4.3 — Chain of Custody Writeup

`grc-gate.yml` produces a signed, retained evidence bundle on every run. Each of the four chain-of-custody properties below is backed by a specific artifact, not a policy statement.

---

## Authenticity — who produced this evidence

**Artifact:** `evidence-<run_id>-<sha>.tar.gz.sig.bundle`, verified by [`scripts/verify-evidence.sh`](scripts/verify-evidence.sh) step 2 (`cosign verify-blob`).

Cosign signs the bundle keylessly using the GitHub Actions OIDC token as the signing identity — no long-lived signing key exists anywhere. Verification pins `--certificate-oidc-issuer https://token.actions.githubusercontent.com`, so a successful verify proves the bundle was signed by *this* workflow running in GitHub Actions, not by an arbitrary laptop or process holding a leaked key.

## Integrity — the bytes haven't changed since signing

**Artifact:** `evidence-<run_id>-<sha>.tar.gz.sha256`, checked by `verify-evidence.sh` step 1.

The SHA-256 digest is computed once, in CI, at the moment the bundle is created, and stored as a sidecar object. Verification recomputes the hash locally and compares. Demonstrated on run `31067345555`: appending a single byte to a local copy of the bundle changed the digest from `543dfe2b...` to `fcd6efe0...`, and both the manual hash comparison and `cosign verify-blob` independently rejected the tampered bytes (`error verifying bundle: matching bundle to payload`, exit 1). The signature is computed over the original bytes; it does not — and cannot — validate a modified file.

## Timeliness — when this evidence was produced

**Artifact:** the Rekor transparency log entry embedded in the `.sig.bundle`, plus `run_id`/`commit` in `receipt.json`.

Cosign's keyless signing writes a timestamped, publicly auditable entry to Sigstore's Rekor log at signing time. That entry — checked implicitly by `cosign verify-blob` — proves the signature (and therefore the bundle) existed at a specific point in time tied to a specific GitHub Actions run and commit SHA, not merely that some file matches some hash with no anchor to *when*.

## Preservation — the evidence can't be quietly destroyed

**Artifact:** S3 Object Lock (`GOVERNANCE` mode, vault bucket `cgep-lab-grc-evidence-vault-8d473e64`), checked by `verify-evidence.sh` step 3 (`aws s3api get-object-retention`).

Every object written to the vault inherits a retention date during which it cannot be deleted or have its retention shortened without `s3:BypassGovernanceRetention`, which the CI role does not hold. This was validated directly: an attempt to overwrite the bundle's key with tampered bytes did **not** destroy the original — S3 versioning meant the tampered write became a new, separate version, while the original clean version remained present and locked (`VersionId: B0DanI2eG0YnMpt7ZRTz5F9xWE3fvDlO`), and was used to restore the key to its clean state.

**Caveat worth stating plainly:** Object Lock protects existing *versions* from deletion, but a versioned bucket still accepts a new `PutObject` to the same key — it does not reject the write outright the way a non-versioned, locked object would. "Latest" can therefore be shadowed by a new version. The durable anchor is the specific `VersionId` recorded at signing time, not the key name alone; a rigorous verifier should pin to that version rather than trust whatever is currently "latest".
