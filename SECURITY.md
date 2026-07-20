# Security Policy

## Reporting a vulnerability

Report security issues privately through GitHub: open the [Security tab](https://github.com/nebinfra/trust/security) of this repository and choose **Report a vulnerability**, or email <security@nebinfra.com>. If you believe this repository itself may be compromised, prefer email.

Please do not open public issues or pull requests for security reports.

This repository is the trust anchor for NebInfra release verification, so reports here are treated at the highest severity. We aim to acknowledge them within 1 business day. We follow coordinated disclosure: we ask that you give us a reasonable window to respond before publishing details. There is no paid bounty program at this time; we are happy to credit reporters on request.

## Scope

This repository publishes the cosign public keys used to verify every NebInfra release artifact. In scope here:

- An unexpected change to any key file in `cosign-keys/`, or history rewrites affecting them.
- A discrepancy between a key published here and the signature on a released artifact.
- Rotation anomalies: keys changing outside the documented quarterly rotation, or archived keys disappearing.
- Any suspected compromise of the private signing keys (the private halves live only in AWS KMS and never leave it).

Vulnerabilities in the released binaries belong in the product's release repository: [nebinfra/nebcli-dist](https://github.com/nebinfra/nebcli-dist) or [nebinfra/nebguard-dist](https://github.com/nebinfra/nebguard-dist).

## Verifying this repository

Keys in this repository can be cross-checked out-of-band: every signature also carries a Rekor transparency-log inclusion proof in its `.bundle`, so a forged key cannot retroactively validate an artifact that was never logged. If a key here fails to verify an artifact that the Rekor log says was signed, or verifies one the log has never seen, report it immediately.

## Key lifecycle

Keys rotate quarterly (the 1st of January, April, July, and October). Previous keys are preserved under `cosign-keys/archive/` so historical releases remain verifiable. A missing or altered archive entry is a security report.
