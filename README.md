# kenshoo-workflows

**Reusable GitHub Actions workflows that build, scan, sign and attest container images.**

The main workflow, `secure-build`, takes a repository with a Dockerfile and produces a multi-architecture image that is scanned, documented with a software bill of materials, signed without long-lived keys, and published with build provenance. It is the first link of the [kenshoo](https://github.com/Kenshoo-ops/kenshoo) supply chain, and it is built to be reused by any other repository.

> **Status: early development.** The workflows are not written yet. The interface below is the intended design and may change before `v1`. See the [roadmap](#roadmap).

## What `secure-build` will do

| Step | Tool | Result |
|---|---|---|
| Build | Docker Buildx | Multi-arch image (amd64 and arm64) |
| Scan | Trivy | The build fails on unreviewed critical vulnerabilities |
| SBOM | Syft | A software bill of materials for the image |
| Sign | cosign (keyless, GitHub OIDC) | A signature tied to this workflow's identity, with no key to leak |
| Attest | GitHub attestations | Verifiable record of how and where the image was built |
| Publish | GitHub Container Registry | Image, signature and SBOM stored together |

## Planned usage

*This is the intended interface. Names and inputs may change before the first release.*

```yaml
jobs:
  build:
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    uses: Kenshoo-ops/kenshoo-workflows/.github/workflows/secure-build.yml@v1
    with:
      image: ghcr.io/<owner>/<image-name>
      runner: ubuntu-24.04   # optional, which runner to use
```

Always pin to a release tag (`@v1`) rather than `main`, so a change here can never break your pipeline without you opting in.

## Verifying an image

Anyone, including the cluster's admission policy, can check that an image was built by this pipeline:

```bash
cosign verify ghcr.io/<owner>/<image-name>:<tag> \
  --certificate-identity-regexp '^https://github.com/<owner>/<repo>/' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

*Check the exact identity values against the first release, because they depend on how the workflow is called.*

## Design principles

- **No long-lived secrets.** Signing uses short-lived identity-based certificates.
- **Least privilege.** Each workflow requests only the permissions it needs.
- **Pinned and versioned.** Releases use semantic version tags, and third-party actions are pinned.
- **The runner is a parameter.** Workflows run on GitHub-hosted runners by default and accept a different runner as an input.

## Roadmap

- [ ] `secure-build` v0.1: build, scan, SBOM, sign
- [ ] Provenance attestation
- [ ] Tested against a sample repository
- [ ] `v1.0.0` release with documented inputs and outputs
- [ ] Additional workflows (dependency review, secret scanning, release automation)

## Part of a larger project

| Repository | Purpose |
|---|---|
| [kenshoo](https://github.com/Kenshoo-ops/kenshoo) | The backup verifier, written in Go |
| [kenshoo-platform](https://github.com/Kenshoo-ops/kenshoo-platform) | Infrastructure and security policies |
| **kenshoo-workflows** (this repo) | The reusable CI pipeline |

## Contributing

The project is just getting started. Issues and ideas are welcome. Please open an issue before starting a large change. All changes go through pull requests.

## Security

Please report vulnerabilities privately. See [SECURITY.md](SECURITY.md).

## License

Apache License 2.0. See [LICENSE](LICENSE).
