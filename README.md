# image-mirror

Mirrors pinned upstream container image digests into this organization's GHCR
namespace, preserving digests and attaching build provenance.

## Why

Upstream registries make no durability promise for manifests that are no longer
tagged. When an image is pinned by digest and the upstream tag later moves, that
manifest becomes *untagged* — and whether it survives is registry **policy, not
contract**.

The failure mode is unpleasant because it is delayed and silent. A pinned
deployment keeps working indefinitely on cached layers, then fails the first time
a node is rebuilt or a pod is scheduled somewhere new — which tends to be during
a reinstall or a recovery, i.e. the worst possible moment to discover an image
has evaporated.

Mirroring copies the exact manifest into a namespace we control, on a tag we
never move.

## Properties

- **Digest-preserving.** `crane copy` transfers manifest bytes verbatim, so a
  mirrored image has the **same digest** as upstream. Consumers change only the
  registry host — the pinned digest itself never moves. The workflow asserts
  source and destination digests match and fails the run if they ever diverge.

- **Multi-arch safe.** The full manifest index is copied. This is the main reason
  to use `crane` rather than `docker pull` + `docker push` from a workstation:
  the latter silently resolves to the *local* machine's architecture, so an
  arm64 laptop would publish an arm64-only image that breaks amd64 consumers.

- **Attested.** Each mirrored digest receives a SLSA build-provenance
  attestation, verifiable via `gh attestation verify --bundle-from-oci`.

  Be precise about what this asserts: the attestation records that **this
  workflow placed that digest here**. It does not — and cannot — vouch for how
  upstream built the image. If upstream publishes its own signature, verify that
  separately against the upstream identity.

## Adding a mirror

Add an entry to `matrix.include` in `.github/workflows/build.yml`:

```yaml
- src: <upstream-repo>@sha256:<digest>
  dst_repo: ghcr.io/<org>/<name>
  dst_tag: <immutable-tag>
```

Two rules:

1. **`src` must be digest-pinned.** Mirroring a floating tag copies whatever
   happens to be there at run time, which defeats the entire purpose.
2. **`dst_tag` must be immutable** — never reuse a tag for different content.
   Encoding the upstream provenance is a good default: a release version, or the
   upstream commit sha the build came from.

Push to `main` and the workflow runs. Copies are idempotent, so re-running an
unchanged entry is a no-op.

## Making a mirrored package pullable

New GHCR packages are **private by default**. Either set the package visibility
to Public in its settings, or give consumers a pull secret.

Beware a misleading check here: **GHCR returns `401` to unauthenticated manifest
requests even for public images.** Anonymous clients must first obtain a token:

```sh
TOKEN=$(curl -s "https://ghcr.io/token?scope=repository:<org>/<name>:pull&service=ghcr.io" \
  | sed -n 's/.*"token":"\([^"]*\)".*/\1/p')
curl -sI -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/vnd.oci.image.index.v1+json" \
  "https://ghcr.io/v2/<org>/<name>/manifests/<tag>"
```

A bare unauthenticated `curl -I` is **not** a valid test of whether a package is
public — it returns `401` either way.

## Verifying a mirror

```sh
gh attestation verify oci://ghcr.io/<org>/<name>:<tag> \
  --owner <org> --bundle-from-oci
```

Use `--bundle-from-oci`: it exercises the **registry** fetch path — the same path
an admission controller such as Kyverno would use — which can behave differently
from the API path. A verification that passes via the API but fails from the
registry is a false green.

Worth confirming the check can actually fail: run it once with a wrong
`--owner` and make sure it exits non-zero. A verifier that passes for any input
proves nothing.

Note that `gh attestation verify` prints nothing on success in a non-interactive
shell — check the exit code, not the output.

## Retiring a mirror

Remove the matrix entry once no consumer references it, then delete the package
version. Leaving stale entries means continuing to publish, and implicitly
vouch for, images nothing uses.

## License

MIT — see [LICENSE](LICENSE).
