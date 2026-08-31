<!-- Last-Updated: 2026-08-25T00:00:00Z -->
# Security Matrix — 2.2.x Stream

## Version Stream

This Konflux release repo covers the **2.2.x** product version stream.

## Supportability Matrix

| RHTPA Version | Build | Build Date | trustify | trustify-ui | Notes |
|---|---|---|---|---|---|
| 2.2.0 | 0.4.5 | 2025-12-03 | `v0.4.5` | `bb447eeb9f7f3b6336ad7ef4e46322b8ca44d6a0` | |
| 2.2.1 | 0.4.8 | 2026-02-05 | `v0.4.8` | `40874efb8a230f871419e6383d8eeba00eba7748` | |
| 2.2.2 | 0.4.9 | 2026-02-23 | `v0.4.9` | `40874efb8a230f871419e6383d8eeba00eba7748` | trustify-ui retag of 2.2.1 |
| 2.2.3 | 0.4.11 | 2026-03-23 | `v0.4.11` | `bf9fc536b33d5223c61d13600230ee5e01ba9bd5` | |
| 2.2.4 | 0.4.12 | 2026-05-04 | `v0.4.12` | `9e03dca1496e42c28be75b5e6f797854c3857bdd` | |
| 2.2.5 | 0.4.13 | 2026-06-02 | `v0.4.13` | `618fa8c57f6314c254da600b8312916fbf98c7bb` | |
| 2.2.6 | 0.4.14 | 2026-06-22 | `e742803b7a76` | `618fa8c57f6314c254da600b8312916fbf98c7bb` | source pinning switched to git submodule |
| 2.2.7 | 0.4.18 | 2026-08-24 | `75ad0566fd90` | `885712be849f4fa90e0122bed33ae8213bb3b82d` | |

### Source Pinning Method

- **trustify**: `artifacts.lock.yaml` for builds ≤ 0.4.13 (download URL contains tag, e.g., `v0.4.13`); git submodule at `components/trustify` for builds ≥ 0.4.14 (internal mirror `trustification/trustify-guacsec.git`, branch `release/0.4.z`)
- **trustify-ui**: git submodule at `components/trustify-ui`

## Ecosystem Mappings

| Ecosystem | Repository | Lock File | Check Command | Upstream Branch |
|---|---|---|---|---|
| Cargo | trustify | `Cargo.lock` | `git show <tag>:Cargo.lock \| grep -A2 'name = "<package>"'` | `release/0.4.z` |
| npm | trustify-ui | `package-lock.json` | `git show <commit>:package-lock.json \| python3 -c "import sys,json; d=json.load(sys.stdin); pkgs=d.get('packages',d.get('dependencies',{})); [print(f'{p}: {i.get(\"version\",\"?\")}') for p,i in pkgs.items() if '<package>' in p.split('/')[-1]]"` | `release/0.4.z` |

## Forward Pointer

Next stream: `3.0.x` at `gitlab.cee.redhat.com/trustification/rhtpa-konflux-release.0.5.z`
