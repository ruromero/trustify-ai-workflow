<!-- Last-Updated: 2026-11-16T15:08:14Z -->
# Security Matrix — 3.0.x Stream

## Version Stream

This Konflux release repo covers the **3.0.x** product version stream.

## Supportability Matrix

| RHTPA Version | Build | Build Date | trustify | trustify-ui | operator | Notes |
|---|---|---|---|---|---|---|
| 3.0.0 | 0.5.0 | 2026-07-16 | `eeca12e2f012fec2c29aa380d1bf04376f637e7f` | `b0b5c5f85428ebc862d3f6979d2dd21a7751517c` | `3.0.0-1784728066` | tag v3.0.0-0.5.0; operator go1.26.5 / `x/net` v0.57.0 — below the CVE-2026-78660 threshold |

The `operator` column records the `3.0.z` series GA operator build (`latest`)
in effect for that product version. The operator image is versioned and released
independently of the product, so its full build history, digests and SBOM
evidence live in `docs/security-matrix-operator.md`. The fixed `3.0.1` build
exists only as a pre-release candidate and is tracked in TC-7586.

### Source Pinning Method

- **trustify**: git submodule at `components/trustify` (internal mirror `trustification/trustify-guacsec.git`, branch `release/0.5.z`)
- **trustify-ui**: git submodule at `components/trustify-ui` (internal mirror `trustification/trustify-ui-guacsec.git`, branch `release/0.5.z`)

## Ecosystem Mappings

| Ecosystem | Repository | Lock File | Check Command | Upstream Branch |
|---|---|---|---|---|
| Cargo | trustify | `Cargo.lock` | `git show <tag>:Cargo.lock \| grep -A2 'name = "<package>"'` | `main` |
| npm | trustify-ui | `package-lock.json` | `git show <commit>:package-lock.json \| python3 -c "import sys,json; d=json.load(sys.stdin); pkgs=d.get('packages',d.get('dependencies',{})); [print(f'{p}: {i.get(\"version\",\"?\")}') for p,i in pkgs.items() if '<package>' in p.split('/')[-1]]"` | `main` |

## Forward Pointer

Next stream: `3.1.x` at `gitlab.cee.redhat.com/trustification/rhtpa-konflux-release.0.6.z`
