<!-- Last-Updated: 2026-11-16T15:08:14Z -->
# Security Matrix — Operator Streams (trusted-profile-analyzer-operator)

## Version Stream

The operator is built, signed and released as its **own Konflux application**
from `trustification/trusted-profile-analyzer-operator`. It is not part of the
`rhtpa-product-*` release repo, does not use its submodule pinning, and follows
its own version series. As of 2026-11-16 each stream has a **dedicated Konflux
application** built from its own release branch (the shared `operator-3-y-z`
application was purged on 2026-10-29, commit `3522785`):

| Operator series | Ships with product stream | Konflux app / branch | Image repository |
|---|---|---|---|
| `1.1.z` | RHTPA 2.2.x | `operator-1-1-z` / `release/1.1.z` | `registry.redhat.io/rhtpa/rhtpa-rhel9-operator` |
| `3.0.z` | RHTPA 3.0.x | `operator-3-0-z` / `release/3.0.z` | `registry.redhat.io/rhtpa/rhtpa-rhel10-operator` |
| `3.1.z` | RHTPA 3.1.x | `operator-3-1-z` / `release/3.1.z` | `registry.redhat.io/rhtpa/rhtpa-rhel10-operator` |
| `3.2.z` | RHTPA 3.2.x | `operator-3-2-z` / `release/3.2.z` | `registry.redhat.io/rhtpa/rhtpa-rhel10-operator` |

Operator CVEs are reported against `pscomponent:rhtpa/rhtpa-rhel9-operator` and
`pscomponent:rhtpa/rhtpa-rhel10-operator`. The product stream matrices carry an
`operator` column that links a product version to the operator build it shipped.

## Supportability Matrix

### Series 1.1.z — RHTPA 2.2.x (`rhtpa/rhtpa-rhel9-operator`)

| Operator Build | Build Date | Image Digest | Go stdlib | golang.org/x/net | Notes |
|---|---|---|---|---|---|
| 1.1.0-1763558499 | 2025-11-19 | `82c7c370c029a8fc6a0a918c16ea14e25aab8aa8b87ca586b31f3aa5e8633b39` | go1.24.6 | v0.42.0 | |
| 1.1.1-1771481125 | 2026-02-19 | `f72d0c4a8338a9145d49315a10191e22c042b0dd26cffbdc797d3fed31742301` | go1.25.5 | v0.47.0 | |
| 1.1.2-1771952936 | 2026-02-24 | `9db50052057ad4ec94099bc3c9ab476f5ac7b00954c99cb0b26fe729abba3387` | go1.25.5 | v0.47.0 | |
| 1.1.3-1774279924 | 2026-03-23 | `cb5b933220b1493a9fd73380d296b3d6f79c594fc28446e0197c9a3b143025ef` | go1.25.7 | v0.48.0 | |
| 1.1.4-1776345412 | 2026-04-16 | `95f5dcecd4c2f488420d608565bcac75909a65f2c9d443c34d93f7914f1cb1ef` | go1.25.8 | v0.48.0 | |
| 1.1.5-1782146014 | 2026-06-22 | `a2c1332d6fe22fed5717cc8b2f3cb1d533a1af0463ade1595c770b41fb3cff26` | go1.26.3 | v0.48.0 | |
| 1.1.6-1782811156 | 2026-06-30 | `229a5afbbf57f509956f87aebdfffb93b97eda8811ebdec91815713da6811b59` | go1.26.3 | v0.48.0 | |
| 1.1.7-1784194900 | 2026-07-16 | `83e44e681aa6effc07db156d83aa54b184221c1ae5a21cef1806e15631c54b54` | go1.26.5 | v0.57.0 | |
| 1.1.8-1788879257 | 2026-09-08 | `69a646cc24828f652248c93da576b7b0ce5ccccef9b21e8be7bc13ed7c506016` | go1.26.7 | v0.58.0 | matches `release/1.1.z` HEAD (go 1.26.7 / x/net v0.58.0) |
| 1.1.9-1793189136 | 2026-10-28 | `b2233c3cfc5f12e014cbe3e2fd578ea5e6cc69f6298e2c31dbf51be9146e249c` | go1.27.3 | v0.61.0 | current `1.1.9` / `latest`; `release/1.1.z` HEAD (`ba90c59`, 2026-11-13) is go 1.27.3 / `x/net` v0.62.0, so provenance is consistent |

### Series 3.0.z / 3.1.z — RHTPA 3.0.x / 3.1.x (`rhtpa/rhtpa-rhel10-operator`)

| Operator Build | Stream | Build Date | Image Digest | Go stdlib | golang.org/x/net | Notes |
|---|---|---|---|---|---|---|
| 3.0.0-1784728066 | 3.0.x | 2026-07-22 | `259595deb9a54fb3b90670c1ac350bb29e87e3afd22840ecb215fce00b2b6921` | go1.26.5 | v0.57.0 | only 3.0.x release; still the stream's GA image |
| 3.1.0-1786625268 | 3.1.x | 2026-08-13 | `e2f28eef2f9a53c0321d7eece236cc2bbdf28937c61f16f6cad54e930022a583` | go1.26.5 | v0.57.0 | |
| 3.1.1-1791291384 | 3.1.x | 2026-10-06 | `b6bcdcd895c32594b58721224b3c29321aa669dc73a66d01f81becf24beadcb5` | go1.26.7 | v0.59.0 | first 3.1.1 build |
| 3.1.1-1791366997 | 3.1.x | 2026-10-07 | `c2bd4c8ad667f5519874345b2a8a0014a31a607df46a85040b5a33256dfc2b60` | go1.26.7 | v0.59.0 | second 3.1.1 build; current `3.1.1` / `latest` |

### Unreleased (RC) builds — fixed operator candidates

Built in the Konflux tenant namespace and **not yet released** to
`registry.redhat.io`. Tags follow the `v<version>-rc` convention on the
`trusted-content-tenant` repositories.

| Candidate | Stream | Build Date | Source commit | Manifest Digest | Bundle pin | Go stdlib | golang.org/x/net |
|---|---|---|---|---|---|---|---|
| `operator-3-0-z:v3.0.1-rc` | 3.0.x | 2026-11-13 | `282332aa` | `sha256:7c03ef74f61ff3a9ba3a1a8b38c80c585888f0a5676f117ed8ff0ae92776db89` | `sha256:7c03ef74…76db89` (same build) | go1.27.3 | v0.62.0 |
| `operator-3-1-z:v3.1.2-rc` | 3.1.x | 2026-11-12 | `724b9906` | `sha256:4779053e3181aeda04bf0ce76b9366e50b302e73a405987b8e796c2d18397a88` | `sha256:a0d7f319…04409` (same commit, earlier build) | go1.27.3 | v0.62.0 |
| `operator-1-1-z:v1.1.9-rc` | 2.2.x | 2026-11-13 | `6abf67a9` | `sha256:5212367a0b605cbdab3ce02ae7c8768d81b4220b8c50b1e615b286ff178560de` | `sha256:2f875ee3…3fa23` | go1.27.3 | v0.62.0 |

The **Bundle pin** column is the manager image digest referenced by
`bundle/manifests/rhtpa-operator.clusterserviceversion.yaml` on the release
branch (`release/3.0.z` HEAD `436a315`, `release/3.1.z` HEAD `90812fc`, both
verified 2026-11-16). On 3.0.z the pin equals the `v3.0.1-rc` build; on 3.1.z it
is a second build of the same commit `724b9906` (`a0d7f319…`, 2026-11-12T20:57Z)
rather than the tagged `v3.1.2-rc` digest (`4779053e…`, 21:10Z).

All three release branches are already above the CVE-2026-78660 thresholds
(Go >= 1.26.9 / >= 1.27.2, `x/net` >= 0.60.0). On 3.0.x and 3.1.x the branch
bundle already pins a fixed digest, so no source or bundle change remains — the
outstanding work is GA release propagation only, tracked in TC-7585 (3.1.x) and
TC-7586 (3.0.x). Neither `3.1.2` nor `3.0.1` is published yet: as of 2026-11-16
`registry.redhat.io/rhtpa/rhtpa-rhel10-operator` still tops out at
`3.1.1-1791366997` and `3.0.0-1784728066`.

### Source Pinning Method

- **Pinning**: image tag `<operator-version>-<unix-build-timestamp>` plus the
  image digest. The tag suffix is the Konflux build timestamp (UTC).
- **No source lock file exists in the image.** The operator is compiled Go; the
  only dependency evidence is the image SBOM, which records each binary's Go
  buildinfo (`sourceInfo: acquired package info from go module information: /manager`).
- **Evidence retrieval**:
  `cosign download sbom registry.redhat.io/rhtpa/<repository>:<tag>`
- **Field extraction**:
  - `stdlib` → `versionInfo` (e.g. `go1.27.3-X:nodwarf5`) and
    `cpe:2.3:a:golang:go:<version>`; strip the `-X:*` suffix to compare against
    Go release versions.
  - `golang.org/x/net` → `versionInfo` (e.g. `v0.61.0`).
  - `pkg:golang/...` purls carry the module graph (1366 entries for 1.1.9).
- **`go.mod` is not authoritative for a shipped image**: branch HEAD can be ahead
  of (or behind) the build that was actually released. As of 2026-11-16 every
  release branch (`release/1.1.z`, `release/3.0.z`, `release/3.1.z`,
  `release/3.2.z`) is at `go 1.27.3` / `golang.org/x/net v0.62.0`, while the
  released 3.x images still carry go1.26.x. Always triage on the image SBOM, not
  on branch HEAD.

## Ecosystem Mappings

| Ecosystem | Repository | Lock File | Check Command | Upstream Branch |
|---|---|---|---|---|
| Go | trusted-profile-analyzer-operator (series 1.1.z) | — (none; image SBOM) | `cosign download sbom registry.redhat.io/rhtpa/rhtpa-rhel9-operator:<tag> \| python3 -c "import json,sys; d=json.load(sys.stdin); [print(p['name'], p.get('versionInfo')) for p in d['packages'] if p['name'] in ('stdlib','golang.org/x/net')]"` | `release/1.1.z` |
| Go | trusted-profile-analyzer-operator (series 3.0.z) | — (none; image SBOM) | `cosign download sbom registry.redhat.io/rhtpa/rhtpa-rhel10-operator:<tag> \| python3 -c "import json,sys; d=json.load(sys.stdin); [print(p['name'], p.get('versionInfo')) for p in d['packages'] if p['name'] in ('stdlib','golang.org/x/net')]"` | `release/3.0.z` |
| Go | trusted-profile-analyzer-operator (series 3.1.z) | — (none; image SBOM) | `cosign download sbom registry.redhat.io/rhtpa/rhtpa-rhel10-operator:<tag> \| python3 -c "import json,sys; d=json.load(sys.stdin); [print(p['name'], p.get('versionInfo')) for p in d['packages'] if p['name'] in ('stdlib','golang.org/x/net')]"` | `release/3.1.z` |
| Go | trusted-profile-analyzer-operator (series 3.2.z) | — (none; image SBOM) | `cosign download sbom registry.redhat.io/rhtpa/rhtpa-rhel10-operator:<tag> \| python3 -c "import json,sys; d=json.load(sys.stdin); [print(p['name'], p.get('versionInfo')) for p in d['packages'] if p['name'] in ('stdlib','golang.org/x/net')]"` | `release/3.2.z` |

## Known Gaps

- The **product version → operator build** mapping is not recorded in any
  release repo: the `rhtpa-konflux-release.0.4.z` repo pins only
  `components/trustify` and `components/trustify-ui`. The `operator` column in
  each product matrix stays `—` until the operator team confirms the mapping.
- **Released 3.x operator images are below the CVE-2026-78660 thresholds.**
  `registry.redhat.io/rhtpa/rhtpa-rhel10-operator` currently publishes only
  `3.0.0-1784728066` (go1.26.5 / `x/net` v0.57.0) and `3.1.1-1791366997`
  (go1.26.7 / `x/net` v0.59.0). Fixed builds exist only as `-rc` candidates
  (see *Unreleased (RC) builds*); GA propagation is tracked in TC-7585 (3.1.x)
  and TC-7586 (3.0.x).
- **RC candidate tags are mutable.** `v<version>-rc` in the
  `trusted-content-tenant` namespace is rebuilt on every push to the release
  branch, so always pin triage evidence to the manifest digest, not the tag.
- The operator always runs as a **second, independent** image in a stream; a
  product image SBOM clean of Go packages says nothing about the operator image.

## Forward Pointer

None — the operator series track the product streams listed above.
