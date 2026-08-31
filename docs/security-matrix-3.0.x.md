<!-- Last-Updated: 2026-08-25T00:00:00Z -->
# Security Matrix — 3.0.x Stream

## Version Stream

This Konflux release repo covers the **3.0.x** product version stream.

## Supportability Matrix

| RHTPA Version | Build | Build Date | trustify | trustify-ui | Notes |
|---|---|---|---|---|---|
| 3.0.0 | 0.5.0 | 2026-07-16 | `eeca12e2f012fec2c29aa380d1bf04376f637e7f` | `b0b5c5f85428ebc862d3f6979d2dd21a7751517c` | tag v3.0.0-0.5.0 |

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
