# Security

## What this repository does and does not assert

This repository copies container images between registries without modifying
them. Every mirrored image is byte-identical to its upstream source — the copy
step asserts the source and destination digests match and fails otherwise.

The SLSA provenance attestation attached to each mirrored digest asserts **only**
that this repository's workflow placed that digest in the destination registry.
It is **not** a statement about how the upstream image was built, what it
contains, or whether it is safe to run. Consumers who need assurance about the
upstream build must verify upstream's own signatures, where they exist, against
the upstream identity.

## Trust boundary

- The workflow authenticates with the per-run `GITHUB_TOKEN`. No long-lived
  registry credential is stored in this repository or its secrets.
- Third-party actions are pinned to full commit SHAs, not tags, so a moved tag
  upstream cannot change what runs here.
- Every `src` must be digest-pinned. A floating source tag would let upstream
  change what gets mirrored after review.

## Reporting a problem

Open an issue, or contact the repository owner directly for anything that
should not be discussed publicly.
