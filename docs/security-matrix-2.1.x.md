<!-- Last-Updated: 2026-08-25T00:00:00Z -->
# Security Matrix — 2.1.x Stream (EOL)

## Version Stream

This Konflux release repo covers the **2.1.x** product version stream. **This stream has reached End of Life (EOL) and will not receive further updates.**

## Supportability Matrix

| RHTPA Version | Build | Build Date | trustify | trustify-ui | Notes |
|---|---|---|---|---|---|
| 2.1.0 | 0.3.5 | 2025-07-16 | `0a49e22bf2efc4313de294f985654cd69c58c491` | `c58347d355f3e15ee8bdf0810e4f76c448941ae2` | |
| 2.1.1 | 0.3.6fix | 2025-09-09 | `9145a3e7d756938815f59251011f95f7412e45a3` | `e435a7f67d5278c264952c30b9e76cf33dfc0113` | |

### Source Pinning Method

- **trustify**: git submodule at `components/trustify`
- **trustify-ui**: git submodule at `components/trustify-ui`

## Ecosystem Mappings

| Ecosystem | Repository | Lock File | Check Command | Upstream Branch |
|---|---|---|---|---|
| Cargo | trustify | `Cargo.lock` | `git show <commit>:Cargo.lock \| grep -A2 'name = "<package>"'` | `release/0.3.z` |
| npm | trustify-ui | `package-lock.json` | `git show <commit>:package-lock.json \| python3 -c "import sys,json; d=json.load(sys.stdin); pkgs=d.get('packages',d.get('dependencies',{})); [print(f'{p}: {i.get(\"version\",\"?\")}') for p,i in pkgs.items() if '<package>' in p.split('/')[-1]]"` | `release/0.3.z` |

## Forward Pointer

Next stream: `2.2.x` at `gitlab.cee.redhat.com/trustification/rhtpa-konflux-release.0.4.z`
