# Python uv Package Manager -- Parity Report (2026-04-22, updated 2026-04-23)

## Context

PR #428 on `guacsec/trustify-da-java-client` (branch `TC-3855`, from fork `a-oren/trustify-da-java-client`) adds uv package manager support. This report compares it against the existing JS client implementation to identify parity gaps.

**Update 2026-04-23:** Commit `a9174b5` switched the Java provider from `uv pip list/show` to `uv export`, aligning the resolution strategy with the JS client. Critical gaps C1 and C4 are now RESOLVED. Moderate gap M1 is now RELEVANT (editable members are skipped, not resolved). M2 is now RESOLVED (both use `TRUSTIFY_DA_UV_EXPORT`).

---

## 1. Architecture Comparison

| Aspect | Java (PR #428) | JavaScript (main) |
|---|---|---|
| Provider class | `PythonUvProvider extends PythonProvider` | `Python_uv extends Base_pyproject` |
| Factory/dispatch | `PythonProviderFactory.create()` checks `uv.lock` in manifest dir | Provider array in `provider.js:32-34`, priority: Poetry > uv > pip |
| Shared utilities | `PyprojectTomlUtils` (new, extracted) | `Base_pyproject` (inherited methods) |
| Resolution strategy | `uv export --format requirements.txt --frozen --no-hashes --no-dev --no-emit-project` | `uv export --format requirements.txt --frozen --no-hashes --no-dev --no-emit-project` |
| Graph construction | Forward graph from `# via` comments in export output | Inverted graph from `# via` comments in export output |
| Parsing | Text parsing of `# via` comments | tree-sitter (`tree-sitter-requirements.wasm`) + text (`# via`) |

### Resolution Strategy: ALIGNED (as of 2026-04-23)

Both implementations now use the same `uv export` command. The parsing approaches differ slightly (Java uses plain text splitting, JS uses tree-sitter for the package lines plus text parsing for `# via` comments), but the effective behavior is equivalent.

---

## 2. Feature-by-Feature Comparison

### 2.1 Lock File Detection

| Feature | Java (PR #428) | JS | Parity |
|---|---|---|---|
| Lock file name | `uv.lock` | `uv.lock` | MATCH |
| Detection location | `PythonProviderFactory.create()` checks manifest dir | `Base_pyproject.validateLockFile()` checks via `_findLockFileDir()` | See walk-up below |
| Validation method | `validateLockFile()` throws `IllegalStateException` if missing | `validateLockFile()` returns boolean (false if not found) | **DIFFERENT** -- Java throws, JS returns false |

### 2.2 Lock File Walk-up (Workspace Support)

| Feature | Java (PR #428) | JS | Parity |
|---|---|---|---|
| Walk-up to find `uv.lock` | **NO** -- only checks manifest dir | **YES** -- `_findLockFileDir()` walks up directories | **GAP in Java** |
| Workspace root boundary | **NO** | **YES** -- stops at `[tool.uv.workspace]` | **GAP in Java** |
| `TRUSTIFY_DA_WORKSPACE_DIR` env var | **NO** | **YES** -- overrides walk-up | **GAP in Java** |
| Editable member handling (`-e path/`) | **N/A** (no `uv export`, so no `-e` entries) | **YES** -- reads member `pyproject.toml` for name/version | **GAP in Java** |

### 2.3 Binary Discovery

| Feature | Java (PR #428) | JS | Parity |
|---|---|---|---|
| Default binary | `uv` (via `Operations.getExecutable("uv", "--version")`) | `uv` (via `getCustomPath('uv', opts)`) | MATCH |
| Custom path env var | `TRUSTIFY_DA_UV_PATH` (via `Operations.getCustomPathOrElse()`) | `TRUSTIFY_DA_UV_PATH` (via `getCustomPath()` -> `getCustom()`) | **MATCH** |
| Binary validation | Runs `uv --version` at construction time; throws if not found | No explicit validation; fails on first use | Java is stricter |

### 2.4 Dependency Resolution

| Feature | Java (PR #428, post-refactor) | JS | Parity |
|---|---|---|---|
| CLI command | `uv export --format requirements.txt --frozen --no-hashes --no-dev --no-emit-project` | `uv export --format requirements.txt --frozen --no-hashes --no-dev --no-emit-project` | **MATCH** |
| Requires installation | **NO** (reads from lock file) | **NO** (reads from lock file) | **MATCH** |
| Dev dependency exclusion | **YES** (`--no-dev` flag) | **YES** (`--no-dev` flag) | **MATCH** |
| Self-reference exclusion | **YES** (`--no-emit-project` flag) | **YES** (`--no-emit-project` flag) | **MATCH** |
| Direct dep identification | Reads `# via` comments; deps via project name = direct | Reads `# via` comments; deps via project name = direct | **MATCH** |
| Transitive graph | From `# via` comments in `uv export` output | From `# via` comments in `uv export` output | **MATCH** |
| Editable member handling (`-e path/`) | Skips lines starting with `-e ` (no resolution) | Reads member `pyproject.toml` for name/version | **GAP in Java** |
| Env var override for testing | `TRUSTIFY_DA_UV_EXPORT` (plain text) | `TRUSTIFY_DA_UV_EXPORT` (base64 encoded) | MATCH (name); encoding differs (plain vs base64) |

### 2.5 PEP 508 Extras Stripping

| Feature | Java (PR #428) | JS | Parity |
|---|---|---|---|
| Extras stripping (e.g., `requests[security]` -> `requests`) | **YES** -- `PythonControllerBase.getDependencyName()` strips `[...]` | **NOT NEEDED** -- `uv export` output already strips extras | MATCH (effective behavior) |

### 2.6 Ignore Marker Support

| Feature | Java (PR #428) | JS | Parity |
|---|---|---|---|
| `exhortignore` | **YES** (via `PyprojectTomlUtils.collectIgnoredDeps()` -> `IgnorePatternDetector`) | **YES** (via `Base_pyproject._getIgnoredDeps()` -> `IGNORE_MARKERS`) | MATCH |
| `trustify-da-ignore` | **YES** (via `IgnorePatternDetector.containsIgnorePattern()`) | **YES** (via `IGNORE_MARKERS = ['exhortignore', 'trustify-da-ignore']`) | MATCH |
| Scanning scope | `[project.dependencies]` only | All lines (PEP 621 + Poetry patterns) | JS is broader |
| Ignore in stack mode | Removes from SBOM via `handleIgnoredDependencies()` | Removes from reachability graph via `_reachableNodes()` | **DIFFERENT** mechanism |
| Transitive orphan removal | Java's `handleIgnoredDependencies()` in base class -- depends on SBOM `BelongingCondition.PURL` in "sensitive" mode | JS `_reachableNodes()` does BFS excluding ignored, so unreachable transitives are dropped | **LIKELY MATCH** (Java's "sensitive" belonging condition should produce same effect) |

### 2.7 Root Component Metadata

| Feature | Java (PR #428) | JS | Parity |
|---|---|---|---|
| Root name source | `project.name` from TOML | `parsed.project?.name` | MATCH |
| Root name fallback | `"default-pip-root"` (from `PythonProvider`) | `"default-pip-root"` (from `Base_pyproject`) | MATCH |
| Root version source | `project.version` from TOML | `parsed.project?.version` | MATCH |
| Root version fallback | `"0.0.0"` (from `PythonProvider`) | `"0.0.0"` | MATCH |

### 2.8 License Reading

| Feature | Java (PR #428) | JS | Parity |
|---|---|---|---|
| `project.license` | YES (via `PyprojectTomlUtils.getLicense()`) | YES (via `Base_pyproject.readLicenseFromManifest()`) | MATCH |
| `project.license.text` (PEP 639) | YES | YES (when license is object) | MATCH |
| `tool.poetry.license` fallback | **NO** | YES | JS reads broader (intentional -- Poetry projects) |
| LICENSE file fallback | YES (`LicenseUtils.readLicenseFile()`) | YES | MATCH |

### 2.9 Poetry Rejection

| Feature | Java (PR #428) | JS | Parity |
|---|---|---|---|
| Behavior when `[tool.poetry.dependencies]` exists | Throws `IllegalStateException` | N/A (Poetry provider handles it before uv is tried) | MATCH (by design) |

### 2.10 Name Canonicalization

| Feature | Java (PR #428) | JS | Parity |
|---|---|---|---|
| Algorithm | `name.toLowerCase().replaceAll("[-_.]+", "-")` | `name.toLowerCase().replace(/[-_.]+/g, '-')` | MATCH |

---

## 3. SBOM Structure

| Aspect | Java (PR #428) | JS | Parity |
|---|---|---|---|
| Format | CycloneDX JSON | CycloneDX JSON | MATCH |
| Content type | `application/vnd.cyclonedx+json` | `application/vnd.cyclonedx+json` | MATCH |
| PURL type | `pypi` | `pypi` | MATCH |
| Stack mode | Root -> direct deps -> transitive tree | Root -> direct deps -> transitive tree | MATCH |
| Component mode | Root -> direct deps only (flat) | Root -> direct deps only (flat) | MATCH |

---

## 4. Test Coverage Comparison

| Test Area | Java (PR #428) | JS | Parity |
|---|---|---|---|
| Factory/dispatch | `test_ecosystem_resolves_pyproject_toml_with_uv_lock`, `test_factory_selects_uv_provider_with_lock`, `test_ecosystem_resolves_pyproject_toml_without_uv_lock`, `test_factory_falls_back_to_pyproject_without_lock` | `validateLockFile` suite (uv/poetry cross-check) | MATCH |
| Lock validation | `test_validate_lock_file_passes_with_uv_lock`, `test_validate_lock_file_throws_without_uv_lock` | `validateLockFile` returns boolean | MATCH |
| pip list parsing | `test_parseUvPipList_parses_packages` | N/A (uses tree-sitter for export parsing) | N/A |
| pip show parsing | `test_parseUvPipShow_builds_children` | N/A (uses `# via` comments) | N/A |
| Graph building | `test_buildDependencyGraph_identifies_direct_deps` | Implicit in SBOM comparison tests | MATCH |
| Root metadata | `test_getRootComponentName_reads_pep621_name`, `test_getRootComponentVersion_reads_pep621_version` | Implicit in SBOM comparison | MATCH |
| Stack SBOM | `test_provideStack_with_uv_pip` | `uv_lock` fixture + expected SBOM comparison | MATCH |
| Component SBOM | `test_provideComponent_with_uv_pip` | `uv_lock` fixture + expected SBOM comparison | MATCH |
| Ignore handling | `test_ignored_dependencies_in_uv_project` | `pep621_ignore_and_extras` fixture tests | MATCH |
| Canonicalization | `test_canonicalize` | Implicit | MATCH |
| Workspace walk-up | **NONE** | `uv_workspace` fixture + walk-up test | **GAP in Java** |
| Dev dep exclusion | **NONE** | `uv_dev_deps` fixture + expected SBOM | **GAP in Java** |
| Self-reference exclusion | **NONE** | `uv_self_ref` fixture | **GAP in Java** |

---

## 5. Gap Summary (updated 2026-04-23 after `uv export` refactor)

### Critical Gaps (functional differences that affect correctness)

| # | Gap | Status | Impact | Recommendation |
|---|---|---|---|---|
| ~~C1~~ | ~~Different resolution strategy~~ | **RESOLVED** (commit `a9174b5`) | N/A | Java now uses `uv export` matching JS |
| C2 | **No lock file walk-up** in Java | **OPEN** | uv workspace projects where `uv.lock` is at workspace root (not in member dir) will fail in Java | Add walk-up logic similar to JS `_findLockFileDir()` |
| C3 | **No `TRUSTIFY_DA_WORKSPACE_DIR` support** in Java | **OPEN** | Users cannot override workspace root for uv projects | Add env var check before walk-up |
| ~~C4~~ | ~~No dev dependency exclusion~~ | **RESOLVED** (commit `a9174b5`) | N/A | Java now uses `--no-dev` flag |

### Moderate Gaps (behavioral differences)

| # | Gap | Status | Impact |
|---|---|---|---|
| M1 | **Editable workspace member handling** -- Java skips `-e ` lines entirely; JS reads member `pyproject.toml` for name/version | **OPEN** (now relevant since both use `uv export`) | In uv workspaces, member packages appear as `-e path/to/pkg` in `uv export` output. Java silently drops them, JS resolves them to proper SBOM components. |
| ~~M2~~ | ~~Different env var names for test overrides~~ | **RESOLVED** (commit `a9174b5`) | Both now use `TRUSTIFY_DA_UV_EXPORT`. Encoding differs: Java plain text, JS base64. |
| M3 | **Validation semantics** -- Java `validateLockFile()` throws; JS returns boolean | **ACCEPTED** | Different error handling patterns, consistent with each codebase's conventions. |

### Non-issues (parity confirmed)

- `TRUSTIFY_DA_UV_PATH` env var: Both support it via their standard custom path mechanisms.
- CLI command: Both use `uv export --format requirements.txt --frozen --no-hashes --no-dev --no-emit-project`.
- Dev dependency exclusion: Both use `--no-dev` flag.
- Self-reference exclusion: Both use `--no-emit-project` flag.
- Direct dep identification: Both parse `# via` comments; deps via project name = direct.
- Ignore markers: Both support `exhortignore` and `trustify-da-ignore` in `[project.dependencies]`.
- Name canonicalization: Identical algorithm.
- Root component metadata: Both read `project.name`/`project.version` from pyproject.toml.
- License reading: Equivalent (Java lacks Poetry license fallback but that's appropriate since Java rejects Poetry).
- SBOM format: Identical CycloneDX JSON structure.

---

## 6. Remaining Recommendations (post-refactor)

### Priority 1: Add workspace support (C2, C3)

Now that both clients use `uv export`, workspace support requires:
- `uv.lock` walk-up from manifest directory to find lock file at workspace root
- `[tool.uv.workspace]` boundary detection to stop walk-up
- `TRUSTIFY_DA_WORKSPACE_DIR` env var override
- `PythonProviderFactory.create()` must walk-up to find `uv.lock`, not just check manifest dir

### Priority 2: Handle editable workspace members (M1)

When `uv export` output contains `-e path/to/pkg` entries, the Java provider should read the member's `pyproject.toml` to extract name/version (matching JS `_parseUvExport` lines 71-88) rather than silently skipping them.

### Priority 3: Add test coverage

Add tests for:
- Workspace walk-up (member project with `uv.lock` at parent)
- Editable member resolution
- Workspace boundary detection (`[tool.uv.workspace]`)
