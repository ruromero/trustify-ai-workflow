<!-- Last-Updated: 2026-11-16T15:08:14Z -->
# Security Matrix — 3.1.x Stream

## Version Stream

This Konflux release repo covers the **3.1.x** product version stream.

## Supportability Matrix

| RHTPA Version | Build | Build Date | trustify | trustify-ui | operator | Notes |
|---|---|---|---|---|---|---|
| 3.1.0 | 0.6.0 | 2026-08-03 | `03470672f244` | `eaa52679103a83d73cdc95d6030663726451ecc6` | `3.1.0-1786625268` | tag rhtpa-3.1.0-0.6.0; trustify at v0.6.0-rc.1 |
| 3.1.1 | 0.6.3 | 2026-10-06 | `e399b167a31d` | `67c3823e8f62` | `3.1.1-1791366997` | tag 3.1.1-0.6.3; operator go1.26.7 / `x/net` v0.59.0 — below the CVE-2026-78660 threshold |

The `operator` column records the `3.1.z` series GA operator build (`latest`)
in effect for that product version. The operator image is versioned and released
independently of the product, so its full build history, digests and SBOM
evidence live in `docs/security-matrix-operator.md`. The fixed `3.1.2` build
exists only as a pre-release candidate and is tracked in TC-7585.

### Source Pinning Method

- **trustify**: git submodule at `components/trustify` (internal mirror `trustification/trustify-guacsec.git`, branch `release/0.6.z`)
- **trustify-ui**: git submodule at `components/trustify-ui` (internal mirror `trustification/trustify-ui-guacsec.git`, branch `release/0.6.z`)

## Ecosystem Mappings

| Ecosystem | Repository | Lock File | Check Command | Upstream Branch |
|---|---|---|---|---|
| Cargo | trustify | `Cargo.lock` | `git show <commit>:Cargo.lock \| grep -A2 'name = "<package>"'` | `release/0.6.z` |
| npm | trustify-ui | `package-lock.json` | `git show <commit>:package-lock.json \| python3 -c "import sys,json; d=json.load(sys.stdin); pkgs=d.get('packages',d.get('dependencies',{})); [print(f'{p}: {i.get(\"version\",\"?\")}') for p,i in pkgs.items() if '<package>' in p.split('/')[-1]]"` | `release/0.6.z` |

## Forward Pointer

None — this is the latest development stream.
