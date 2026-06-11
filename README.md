# NebInfra Trust

**Public cryptographic keys and supply-chain transparency for NebInfra production releases.**

This repository is the trust anchor for every container image and binary NebInfra ships to production. The keys here let any third party (operators, customers, auditors, downstream integrators) independently verify that an artifact:

1. Was built by NebInfra's canonical CI/CD pipeline from the canonical source repository.
2. Was signed by a key whose private half lives only inside AWS KMS.
3. Has a publicly auditable record of its signature in the Sigstore Rekor transparency log.

If any of those three checks fail, the artifact is not authentic and must not be run.

> **In scope.** Production release artifacts published from `nebinfra/<app>` repositories.
> **Out of scope.** Non-production builds (PR previews, dev images, staging artifacts) are unsigned by design, so never run them in production.

---

## Table of contents

- [Trust signals at a glance](#trust-signals-at-a-glance)
- [Signed applications](#signed-applications)
- [Quick start: verify a release](#quick-start-verify-a-release)
- [Verification recipes](#verification-recipes)
  - [Container images](#container-images)
  - [Binary release artifacts](#binary-release-artifacts)
  - [SLSA provenance attestations](#slsa-provenance-attestations)
  - [SBOM attestations](#sbom-attestations)
- [Pinning for strict verifiers](#pinning-for-strict-verifiers)
- [Key rotation lifecycle](#key-rotation-lifecycle)
- [Transparency log (Rekor)](#transparency-log-rekor)
- [Architecture and threat model](#architecture-and-threat-model)
- [Reporting concerns](#reporting-concerns)

---

## Trust signals at a glance

| Property | Implementation |
| --- | --- |
| Signing scheme | [Sigstore cosign](https://docs.sigstore.dev/cosign/overview/) with AWS KMS-backed **ECDSA P-256** Customer Master Keys |
| Key custody | Private keys never leave AWS KMS. Signing happens via the KMS API, only from the production CI environment, via a stable alias (`alias/cosign-<app>-prod`) |
| Transparency | Every production signature is published to the public [Sigstore Rekor](https://rekor.sigstore.dev) transparency log |
| Provenance | SLSA-style provenance attestations are recorded with each container image, binding the artifact to its source repository, commit SHA, and the GitHub Actions workflow that built it |
| SBOM | SPDX-JSON SBOM attestations are recorded with each container image |
| Rotation | Quarterly automated rotation on the 1st of January, April, July, October at 00:00 UTC. Old public keys remain available for verification of historical releases |
| Continuous audit | Each `nebinfra/<app>` repository runs a weekly Rekor scan that pages on-call if any signature is found that does not trace back to a legitimate CI run |

---

## Signed applications

| Application | Description | Source repository | Active public key |
| --- | --- | --- | --- |
| `nebguard` | NebCore guardrails engine | [nebinfra/nebguard](https://github.com/nebinfra/nebguard) | [`cosign-keys/nebguard-prod.pub`](cosign-keys/nebguard-prod.pub) |
| `nebcore-cli` | NebCore platform CLI | [nebinfra/nebcore-cli](https://github.com/nebinfra/nebcore-cli) | [`cosign-keys/nebcore-cli-prod.pub`](cosign-keys/nebcore-cli-prod.pub) |
| `nebcore-ai` | NebCore AI runtime | [nebinfra/nebcore-ai](https://github.com/nebinfra/nebcore-ai) | [`cosign-keys/nebcore-ai-prod.pub`](cosign-keys/nebcore-ai-prod.pub) |
| `nebcore-operator` | NebCore Kubernetes operator | [nebinfra/nebcore-operator](https://github.com/nebinfra/nebcore-operator) | [`cosign-keys/nebcore-operator-prod.pub`](cosign-keys/nebcore-operator-prod.pub) |
| `platform-backend` | NebInfra platform API backend | [nebinfra/platform-backend](https://github.com/nebinfra/platform-backend) | [`cosign-keys/platform-backend-prod.pub`](cosign-keys/platform-backend-prod.pub) |

After a rotation, the previous public key is preserved at `cosign-keys/archive/<app>-prod-<YYYY-MM-DD>.pub` so older releases remain verifiable. See [Key rotation lifecycle](#key-rotation-lifecycle).

---

## Quick start: verify a release

You will need [`cosign`](https://docs.sigstore.dev/cosign/installation/) (v2 recommended).

```bash
APP=nebguard
DIGEST=sha256:<digest>          # from your image registry, e.g. `crane digest <image>`
IMAGE=380277571885.dkr.ecr.us-east-2.amazonaws.com/${APP}@${DIGEST}

cosign verify \
  --key https://raw.githubusercontent.com/nebinfra/trust/main/cosign-keys/${APP}-prod.pub \
  --rekor-url https://rekor.sigstore.dev \
  "${IMAGE}"
```

A successful run prints a JSON-encoded verification record. Any non-zero exit code or any output other than that record is a verification failure, so **do not run the artifact**.

---

## Verification recipes

### Container images

```bash
cosign verify \
  --key https://raw.githubusercontent.com/nebinfra/trust/main/cosign-keys/<app>-prod.pub \
  --rekor-url https://rekor.sigstore.dev \
  <image-uri>@sha256:<digest>
```

Always reference images by digest, not by tag. Tags are mutable; digests are not.

### Binary release artifacts

```bash
gh release download v1.2.3 --pattern '*Linux_arm64*' --repo nebinfra/<app>

cosign verify-blob \
  --key https://raw.githubusercontent.com/nebinfra/trust/main/cosign-keys/<app>-prod.pub \
  --rekor-url https://rekor.sigstore.dev \
  --signature <artifact>.sig \
  <artifact>
```

Each release artifact ships with a sibling `<artifact>.sig` file in the same GitHub Release.

### SLSA provenance attestations

```bash
cosign verify-attestation \
  --key https://raw.githubusercontent.com/nebinfra/trust/main/cosign-keys/<app>-prod.pub \
  --rekor-url https://rekor.sigstore.dev \
  --type slsaprovenance \
  <image-uri>@sha256:<digest> | jq '.payload | @base64d | fromjson'
```

The decoded payload exposes:

- `buildType: https://github.com/actions/runner/workflows@v1`: proves the build ran on GitHub Actions.
- `invocation.configSource.uri: git+https://github.com/nebinfra/<app>@refs/heads/main`: proves the build originated from `main` of the canonical NebInfra repository.
- `invocation.configSource.digest.sha1`: pins the exact source commit that produced the artifact.

Strict verifiers should assert all three values match expectations before trusting an image. A mismatch on any of them indicates either a forged signature or a build run from a non-canonical source.

### SBOM attestations

```bash
cosign verify-attestation \
  --key https://raw.githubusercontent.com/nebinfra/trust/main/cosign-keys/<app>-prod.pub \
  --rekor-url https://rekor.sigstore.dev \
  --type spdxjson \
  <image-uri>@sha256:<digest> | jq '.payload | @base64d | fromjson'
```

The SBOM is in SPDX 2.3 JSON format and describes the exact bytes that shipped. Pipe the verified payload into `grype` or `trivy sbom` for vulnerability scanning without re-pulling the image.

---

## Pinning for strict verifiers

Pulling the public key over `https://raw.githubusercontent.com/nebinfra/trust/main/...` trusts whatever currently sits at `main`. A repository takeover could swap a forged key. Strict verifiers should pin the URL to a specific git commit SHA:

```bash
cosign verify \
  --key https://raw.githubusercontent.com/nebinfra/trust/<COMMIT_SHA>/cosign-keys/<app>-prod.pub \
  --rekor-url https://rekor.sigstore.dev \
  <image-uri>@sha256:<digest>
```

Resolve the SHA once and embed it in your verification policy:

```bash
git ls-remote https://github.com/nebinfra/trust main | awk '{print $1}'
```

Re-pin only when a legitimate key rotation is announced via this repository's GitHub Releases.

---

## Key rotation lifecycle

| Step | Action |
| --- | --- |
| **Schedule** | 1st of January, April, July, October at 00:00 UTC |
| **Driver** | The `shared-actions/cosign-key-rotate.yml@v1` reusable GitHub Actions workflow |
| **New key** | A new ECDSA P-256 Customer Master Key is created in AWS KMS |
| **Alias swap** | The stable alias `alias/cosign-<app>-prod` is re-pointed to the new CMK, so production CI immediately uses the new key without code changes |
| **Archive** | The previous public key is moved to `cosign-keys/archive/<app>-prod-<YYYY-MM-DD>.pub` |
| **Activate** | The new public key is committed as `cosign-keys/<app>-prod.pub` |
| **Retire** | The old CMK is scheduled for deletion in AWS KMS with a 30-day recoverability window |

To verify an artifact signed under a previous key, point `--key` at the archived file:

```bash
cosign verify \
  --key https://raw.githubusercontent.com/nebinfra/trust/main/cosign-keys/archive/<app>-prod-<YYYY-MM-DD>.pub \
  --rekor-url https://rekor.sigstore.dev \
  <image-uri>@sha256:<digest>
```

> **Emergency rotation.** If a key is suspected to be compromised, rotation runs out-of-band immediately and the prior key is revoked through a notice in this repository's GitHub Releases. Watch this repository's releases to receive notifications.

---

## Transparency log (Rekor)

Every NebInfra production signature is uploaded to the public Sigstore Rekor transparency log at <https://rekor.sigstore.dev>. This delivers three guarantees that a bare signature does not:

- **Verified timestamping.** Signatures are anchored in time. Verifiers can reject signatures issued outside the expected release window, defeating backdated forgeries.
- **Public auditability.** Anyone, including you, can query Rekor for every entry under a NebInfra public key. Entries that cannot be traced to a legitimate CI run are evidence of key misuse.
- **Continuous monitoring.** Each `nebinfra/<app>` repository runs a weekly `rekor-audit.yml` cron that compares Rekor entries against successful production CI runs and pages on any orphan.

> **Verifier policy must require both a valid signature AND a valid Rekor tlog entry.** This is why every recipe above includes `--rekor-url`. A signature alone proves possession of a private key. A Rekor entry proves the signature was published in the open at a specific point in time.

---

## Architecture and threat model

The full design (KMS topology, CI/CD signing flow, threat model, and incident-response runbooks) is maintained in NebInfra's internal architecture documentation. The verification recipes above are self-contained; no internal document is required to verify artifacts.

---

## Reporting concerns

For suspected key compromise, malicious Rekor entries, supply-chain anomalies, or any other concern about the integrity of a NebInfra release, please open a **security advisory** in the relevant `nebinfra/<app>` repository:

> Repository → **Security** tab → **Advisories** → **Report a vulnerability**

Public issues are also acceptable for non-sensitive reports. Supply-chain reports are treated as the highest priority and routed directly to NebInfra's security on-call.
