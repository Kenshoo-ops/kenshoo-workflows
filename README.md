# kenshoo-workflows

**A reusable GitHub Actions workflow that builds, scans, signs and attests container images.**

`secure-build` takes a repository with a Dockerfile and produces a multi-architecture image that has been vulnerability-scanned, documented with a software bill of materials (SBOM), signed without long-lived keys, and published with verifiable build provenance. Anyone, including a Kubernetes admission policy, can later check where an image came from and what is inside it.

It is the first link of the [kenshoo](https://github.com/Kenshoo-ops/kenshoo) supply chain, and it works for any repository.

> **Status:** `v0.1.1`, early stage. The workflow is tested end to end on a sample repository. Inputs and behavior may still change before `v1.0.0`.

## What `secure-build` does

Each step runs only if the previous one passed.

| # | Step | Tool | Result |
|---|---|---|---|
| 1 | Build and push | Docker Buildx | Multi-arch image (`linux/amd64`, `linux/arm64`) pushed to GitHub Container Registry |
| 2 | Scan | Trivy | The job fails on vulnerabilities at the configured severity |
| 3 | SBOM | Syft (`anchore/sbom-action`) | An SPDX JSON bill of materials for the exact image digest |
| 4 | Sign | cosign (keyless) | A signature tied to this workflow's identity, with no key to protect or leak |
| 5 | Attest SBOM | cosign | The SBOM attached to the image as a signed statement |
| 6 | Attest provenance | GitHub artifact attestations (SLSA) | A signed record of which repository, commit and workflow built the image |

**A failed scan means no signature.** The image is already pushed when the scan runs (multi-arch images cannot be loaded locally first), but signing, the SBOM attestation and provenance never happen. An unsigned image should be rejected by anything that verifies signatures.

## Usage

Call the workflow from a job in your repository:

```yaml
name: ci

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    uses: Kenshoo-ops/kenshoo-workflows/.github/workflows/secure-build.yml@v0.1.1
    with:
      image: ghcr.io/<owner>/<image-name>
```

**The `permissions` block in your calling job is required.** A called workflow can only use permissions its caller grants, so leaving one out makes the run fail.

**The image name must be lowercase**, including the owner part, even if your organization name has capitals.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `image` | yes | | Full image name, for example `ghcr.io/kenshoo-ops/sample-app` |
| `context` | no | `.` | Docker build context |
| `runner` | no | `ubuntu-24.04` | Runner label. GitHub-hosted by default, but any runner label works |
| `severity` | no | `CRITICAL,HIGH` | Vulnerability severities that fail the build |

Trivy only reports vulnerabilities that already have a fix (`ignore-unfixed`). A `.trivyignore` file in the root of the calling repository is applied, so accepted risks can be listed there with a comment explaining why.

### Outputs

| Output | Description |
|---|---|
| `digest` | The digest (`sha256:...`) of the pushed image |

Use the digest, not a tag, when you refer to the image later:

```yaml
jobs:
  build:
    uses: Kenshoo-ops/kenshoo-workflows/.github/workflows/secure-build.yml@v0.1.1
    # ...
  deploy:
    needs: build
    runs-on: ubuntu-24.04
    steps:
      - run: echo "Deploying ghcr.io/<owner>/<image-name>@${{ needs.build.outputs.digest }}"
```

### Image tags

The workflow tags each image with `sha-<short commit>`, the branch name (with `/` replaced by `-`), and the version number when the build runs from a version tag.

### Pin a released version

Always call a release tag such as `@v0.1.1`, not `@main`. A tag keeps your pipeline stable until you choose to update. Dependabot can propose updates if you enable the `github-actions` ecosystem in your repository.

## Verifying an image

You need the image digest. Get it from the workflow output or with:

```bash
docker buildx imagetools inspect ghcr.io/<owner>/<image-name>:<tag> | grep Digest
```

**1. The signature** (`cosign`):

```bash
cosign verify ghcr.io/<owner>/<image-name>@sha256:<digest> \
  --certificate-identity 'https://github.com/Kenshoo-ops/kenshoo-workflows/.github/workflows/secure-build.yml@refs/tags/v0.1.1' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

The identity is the **reusable workflow in this repository**, not your repository. That means one trust rule covers every repository that builds with it. To accept any released version, use `--certificate-identity-regexp` with `'^https://github.com/Kenshoo-ops/kenshoo-workflows/\.github/workflows/secure-build\.yml@refs/tags/v.*'`.

**2. The SBOM** (a signed attestation):

```bash
cosign verify-attestation --type spdxjson ghcr.io/<owner>/<image-name>@sha256:<digest> \
  --certificate-identity-regexp '^https://github.com/Kenshoo-ops/kenshoo-workflows/\.github/workflows/secure-build\.yml@.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

The SBOM is also uploaded to each workflow run as the artifact `sbom.spdx.json`.

**3. The build provenance** (GitHub CLI):

```bash
gh attestation verify oci://ghcr.io/<owner>/<image-name>@sha256:<digest> \
  --owner <owner> \
  --signer-workflow Kenshoo-ops/kenshoo-workflows/.github/workflows/secure-build.yml
```

A successful result names the repository and workflow that built the image and the exact signer workflow version.

**Check that verification can fail.** A verification that never rejects anything proves nothing. Try the same `cosign verify` with a wrong identity, and it must be refused.

## What this does and does not give you

**It gives you:**
- A way to prove an image came from a specific, reviewed workflow
- A list of everything inside the image, signed against tampering
- A scan gate that stops signing when serious vulnerabilities are found
- No signing keys to store, rotate or leak

**It does not give you:**
- A guarantee the application code is correct or free of bugs
- Protection from vulnerabilities that are unknown or have no fix yet
- Anything for registries other than GitHub Container Registry
- Hardened or reproducible builds on its own. Provenance says where an image was built, and does not make the build bit-for-bit reproducible

## Known limitations

- **GitHub Container Registry only.** Other registries are not supported yet.
- **Not meant for pull requests from forks or Dependabot.** Those runs get a restricted token and no `id-token`, so signing fails. Run `secure-build` on pushes to your main branch or on tags.
- **Signatures are written to a public transparency log (Sigstore).** Use it for public software, and do not build images whose contents or names must stay private.
- **Provenance attestations in private repositories may need a paid GitHub plan.** The workflow is tested with public repositories.
- **Actions are pinned by version tag, not commit SHA.** Dependabot keeps them current. Pinning by SHA is planned.

## Design principles

- **No long-lived secrets.** Signing uses short-lived, identity-based certificates.
- **Least privilege.** The workflow requests only the token permissions it needs.
- **Verify, don't trust.** Every artifact can be checked at the point where it is used.
- **The runner is a parameter**, so the workflow does not depend on one provider.
- **Released and versioned.** Releases use semantic version tags that do not move.

## Roadmap

- [x] Multi-arch build and push
- [x] Trivy vulnerability scan gate
- [x] SBOM with Syft
- [x] Keyless signing and signed SBOM attestation
- [x] Build provenance attestation
- [x] Tested end to end on a sample repository
- [ ] Pin third-party actions by commit SHA
- [ ] Separate workflows for pull requests (build and scan only, no signing)
- [ ] Dependency review and secret scanning workflows
- [ ] Release automation
- [ ] `v1.0.0` with stable inputs and outputs

## Part of a larger project

| Repository | Purpose |
|---|---|
| [kenshoo](https://github.com/Kenshoo-ops/kenshoo) | The backup verifier, written in Go |
| [kenshoo-platform](https://github.com/Kenshoo-ops/kenshoo-platform) | Infrastructure, GitOps and admission policies that verify these signatures |
| **kenshoo-workflows** (this repo) | The reusable CI pipeline |
| [sample-app](https://github.com/Kenshoo-ops/sample-app) | A tiny app used to test the pipeline |

## Contributing

Issues and ideas are welcome. Please open an issue before starting a large change. All changes go through pull requests, and a change to `secure-build` should be tested from a caller repository before it is merged.

## Security

Please report vulnerabilities privately. See [SECURITY.md](SECURITY.md).

## License

Apache License 2.0. See [LICENSE](LICENSE).
