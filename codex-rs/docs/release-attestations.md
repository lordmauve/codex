# Release attestations

Codex CLI releases published from this repository include Sigstore-backed [SLSA v1 build-provenance attestations](https://docs.github.com/actions/security-guides/using-artifact-attestations-to-establish-provenance-for-builds). They are generated in the release workflow with [`actions/attest-build-provenance`](https://github.com/actions/attest-build-provenance) and uploaded next to every release artifact as `codex-<target>.bundle.jsonl` files. Each bundle contains the provenance statement for the binary archives built for that target.

The workflow runs on pushed `rust-v*` tags and the attestation is issued to the workflow at `.github/workflows/rust-release.yml`.

## Online verification

Use the GitHub CLI to verify an artifact directly from a release. Replace `<target>` and `<version>` with the values for the asset you downloaded, for example `x86_64-unknown-linux-musl` and `rust-v1.2.3`:

```bash
# Verify a release artifact using GitHub-hosted trust roots
gh attestation verify ./codex-<target>.tar.gz \
  --repo openai/codex \
  --signer-workflow github.com/openai/codex/.github/workflows/rust-release.yml@refs/tags/<version> \
  --source-ref refs/tags/<version> \
  --deny-self-hosted-runners
```

Use `--format json` if you need machine-readable output for a policy engine, or `--predicate-type` to verify a different predicate (for SBOM attestations, for example).

## Offline or air-gapped verification

You can verify the attestation in a disconnected environment by copying a Sigstore bundle and the Sigstore trust roots alongside the binary.

1. From an online machine, download the bundle that was published with the release:

   ```bash
   gh release download <version> \
     --repo openai/codex \
     --pattern "codex-<target>.bundle.jsonl" \
     --dir bundles
   ```

   The download yields `bundles/codex-<target>.bundle.jsonl`, which contains the Sigstore bundle emitted by the workflow.

2. Fetch the Sigstore trust roots that correspond to the GitHub attestation services:

   ```bash
   gh attestation trusted-root > bundles/trusted_root.jsonl
   ```

   Refresh this file periodically online so your offline environment keeps up with trust-anchor rotations.

3. Copy the release artifact, the downloaded bundle, the `trusted_root.jsonl` file, and a recent `gh` CLI binary into the offline environment. Then run:

   ```bash
   gh attestation verify ./codex-<target>.tar.gz \
     --repo openai/codex \
     --bundle codex-<target>.bundle.jsonl \
     --custom-trusted-root trusted_root.jsonl \
     --signer-workflow github.com/openai/codex/.github/workflows/rust-release.yml@refs/tags/<version> \
     --source-ref refs/tags/<version> \
     --deny-self-hosted-runners
   ```

If you store the Sigstore bundle under a different name, update the `--bundle` flag accordingly. The bundle contains all subjects for the target; `gh attestation verify` will match the digest for the artifact you provide.
