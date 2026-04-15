# Python Ecosystem Parity Report -- 2026-04-14

## Executive Summary

The JavaScript client has **significantly broader** Python ecosystem support than the Java client. The JS client supports three pyproject.toml resolution strategies (uv, Poetry, pip fallback) while the Java client only supports pip-based resolution and explicitly rejects Poetry. Neither client has uv support in the Java side.

**Critical parity gaps:**
1. **uv support**: JS only -- Java has zero uv awareness
2. **Poetry support**: JS has full `poetry show` integration -- Java explicitly rejects Poetry with an `IllegalStateException`
3. **Workspace/monorepo support**: JS supports uv workspaces with lock file walk-up -- Java has no workspace concept
4. **Ignore markers in requirements.txt**: JS tree-sitter query only matches `exhortignore`, not `trustify-da-ignore` -- inconsistent with its own pyproject.toml handling and with the Java client

---

## 1. requirements.txt (pip)

### Manifest Detection

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Filename match | `"requirements.txt"` in `Ecosystem.resolveProvider()` | `isSupported('requirements.txt')` in `python_pip.js` | MATCH |
| Provider class | `PythonPipProvider` | `python_pip.js` (default export object) | MATCH (structural difference only) |

### Dependency Resolution

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Resolution strategy | Install packages + `pip freeze` + `pip show` | Install packages + `pip freeze` + `pip show` | MATCH |
| Alternative strategy | `pipdeptree --json` via `TRUSTIFY_DA_PIP_USE_DEP_TREE` | `pipdeptree --json` via `TRUSTIFY_DA_PIP_USE_DEP_TREE` | MATCH |
| Virtual env support | `PythonControllerVirtualEnv` creates venv in `/tmp/trustify_da_env` | `Python_controller` creates venv in `/tmp/trustify_da_env_js` | MATCH (different tmp dir name) |
| Real env support | `PythonControllerRealEnv` uses system python/pip | `Python_controller` with `realEnvironment=true` | MATCH |
| Best-effort install | `TRUSTIFY_DA_PYTHON_INSTALL_BEST_EFFORTS` env var | `TRUSTIFY_DA_PYTHON_INSTALL_BEST_EFFORTS` env var | MATCH |
| Version mismatch check | `MATCH_MANIFEST_VERSIONS` (default true) | `MATCH_MANIFEST_VERSIONS` (default "true") | MATCH |
| Python binary detection | Tries `python3`/`pip3` first, falls back to `python`/`pip` | Tries `python3`/`pip3` first, falls back to `python`/`pip` | MATCH |
| Requirements parsing | Regex/string-based (manual line parsing) | **tree-sitter** (`tree-sitter-requirements.wasm`) | **DIFFERENT** -- same result, different approach |
| Dep name normalization | `replace("-", "_")` and `replace("_", "-")` in cache keys | `replace("-", "_")` and `replace("_", "-")` in cache keys | MATCH |
| Env var overrides for testing | `TRUSTIFY_DA_PIP_SHOW`, `TRUSTIFY_DA_PIP_FREEZE` (base64) | `TRUSTIFY_DA_PIP_SHOW`, `TRUSTIFY_DA_PIP_FREEZE` (base64) | MATCH |
| Circular dependency protection | Path-based loop detection (`!path.contains(dep)`) | Path-based loop detection (`!path.includes(dep)`) | MATCH |
| Env marker handling | Strips markers at `;` before processing | No explicit marker stripping in requirements parsing | **GAP** (see below) |

### Ignore Pattern Handling (requirements.txt)

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Markers recognized | `exhortignore` AND `trustify-da-ignore` | **Only `exhortignore`** | **GAP** |
| Parsing approach | String `contains()` check per line | tree-sitter query: `(#match? @comment "^#[\\t ]*exhortignore")` | Different mechanism |
| Version-aware ignore | Splits `name==version` with regex, passes versioned PURLs | tree-sitter extracts pinned version via `(version_cmp) @cmp (version) @version (#eq? @cmp "==")` | MATCH (behavior) |
| `MATCH_MANIFEST_VERSIONS` interaction | Filters by PURL (versioned) or by name (unversioned) | `filterIgnoredDepsIncludingVersion()` vs `filterIgnoredDeps()` | MATCH |

**GAP DETAIL**: In `requirements_parser.js` line 25, the tree-sitter ignore query hard-codes `exhortignore`:
```
(#match? @comment "^#[\\t ]*exhortignore")
```
This does NOT match `trustify-da-ignore`. The Java client's `PythonProvider.containsIgnorePattern()` checks for both `exhortignore` and `trustify-da-ignore`. The JS `base_pyproject.js` also correctly checks both markers (`IGNORE_MARKERS = ['exhortignore', 'trustify-da-ignore']`), so this is an inconsistency within the JS client itself.

### Environment Marker Handling (requirements.txt)

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Marker-constrained deps | Strips markers at `;`, checks if installed before including | No explicit marker handling in `python_controller.js` | **GAP** |

The Java client (`PythonControllerBase.getDependenciesImpl()`) handles lines like `pywin32>=306; sys_platform == "win32"` by:
1. Splitting at `;` to get the requirement spec
2. Checking if the dep is actually installed in the environment
3. Skipping deps with markers that aren't installed

The Java client has dedicated test fixtures for this: `pip_requirements_txt_marker_skip` and `pip_requirements_txt_marker_installed`.

### SBOM Structure

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Root component name | `"default-pip-root"` | `"default-pip-root"` | MATCH |
| Root component version | `"0.0.0"` | `"0.0.0"` | MATCH |
| PURL type | `pypi` | `pypi` | MATCH |
| Content type | `application/vnd.cyclonedx+json` | `application/vnd.cyclonedx+json` | MATCH |
| Stack analysis | Full tree with transitive deps | Full tree with transitive deps | MATCH |
| Component analysis | Direct deps only (flat) | Direct deps only (flat) | MATCH |
| Dependency sorting | Sorted by name (case-sensitive char comparison) | Sorted by name (case-sensitive) | MATCH |

### Lock File Validation

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Lock file required | No (default `Provider.validateLockFile()` is no-op) | No (`validateLockFile()` returns `true`) | MATCH |

---

## 2. pyproject.toml (PEP 621 / pip fallback)

### Manifest Detection

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Filename match | `"pyproject.toml"` in `Ecosystem.resolveProvider()` | `isSupported('pyproject.toml')` in `Base_pyproject` | MATCH |
| Provider class | `PythonPyprojectProvider` | `Python_pip_pyproject extends Base_pyproject` | MATCH |

### Dependency Resolution

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| CLI command | `pip install --dry-run --ignore-installed --report - .` | `pip install --dry-run --ignore-installed --report - .` | MATCH |
| Pip binary detection | Tries `pip3` then `pip` | Tries `pip3` then `pip` | MATCH |
| Env var override | `TRUSTIFY_DA_PIP_REPORT` (base64) | `TRUSTIFY_DA_PIP_REPORT` (base64) | MATCH |
| Root entry detection | `download_info.dir_info` present | `download_info?.dir_info !== undefined` | MATCH |
| Direct dep extraction | Root's `metadata.requires_dist` | Root's `metadata.requires_dist` | MATCH |
| Extra marker filtering | `hasExtraMarker()` with regex `;\s*.*extra\s*==` | `_hasExtraMarker()` with regex `/;\s*.*extra\s*==/` | MATCH |
| Dep name extraction | Regex `^([A-Za-z0-9]([A-Za-z0-9._-]*[A-Za-z0-9])?)` | Regex `/^([A-Za-z0-9]([A-Za-z0-9._-]*[A-Za-z0-9])?)/` | MATCH |
| Name canonicalization | `name.toLowerCase().replaceAll("[-_.]+", "-")` | `name.toLowerCase().replace(/[-_.]+/g, '-')` | MATCH |
| Transitive graph | Built from each package's `metadata.requires_dist` | Built from each package's `metadata.requires_dist` | MATCH |
| Cycle detection | `visited` Set prevents re-traversal | Not explicitly in `_parsePipReport`; done in `_collectTransitive` via `visited` Set | MATCH |

### Root Component Metadata

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Name source | `project.name` from TOML, fallback to `"default-pip-root"` | `parsed.project?.name`, fallback to `"default-pip-root"` | MATCH |
| Version source | `project.version` from TOML, fallback to `"0.0.0"` | `parsed.project?.version`, fallback to `"0.0.0"` | MATCH |

### License Reading

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| PEP 621 license | `project.license` string | `parsed.project?.license` | MATCH |
| PEP 639 license.text | `project.license.text` | `fromManifest.text` (when license is object) | MATCH |
| Poetry license | NOT read (Poetry rejected) | `parsed.tool?.poetry?.license` | **GAP** -- JS reads Poetry license even though it also has the pip fallback |
| Fallback | `LicenseUtils.readLicenseFile()` | `getLicense(fromManifest, manifestPath)` | MATCH |

### Ignore Pattern Handling (pyproject.toml)

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Markers recognized | `exhortignore` AND `trustify-da-ignore` | `exhortignore` AND `trustify-da-ignore` | MATCH |
| Scanning scope | `[project.dependencies]` array only | All lines (PEP 621 + Poetry style patterns) | JS is broader |
| PEP 621 pattern | Searches for dep string in raw lines + ignore pattern | Regex: `^\s*"([^"]+)"` + ignore marker check | MATCH (behavior) |
| Poetry pattern | N/A (Poetry rejected) | Regex: `^\s*([A-Za-z0-9][A-Za-z0-9._-]*)\s*=` | JS only |
| Version-aware ignore | Always ignores by name (`*` version) | Always ignores by canonical name | MATCH |

### Poetry Rejection

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Behavior | Throws `IllegalStateException` if `[tool.poetry.dependencies]` exists | Does NOT reject -- falls through to pip report | **DIFFERENT** |

The Java `PythonPyprojectProvider.rejectPoetryDependencies()` checks for `[tool.poetry.dependencies]` and throws. The JS `Python_pip_pyproject` has no such check because the provider chain handles this: Poetry projects are matched by `Python_poetry` first (which checks for `poetry.lock`), and `Python_pip_pyproject` only activates as a fallback.

### Lock File Validation

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Lock file required | No | No (always returns true -- fallback provider) | MATCH |

---

## 3. Poetry

### Support Status

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Support level | **REJECTED** -- `IllegalStateException` | **FULLY SUPPORTED** | **CRITICAL GAP** |
| Provider class | N/A | `Python_poetry extends Base_pyproject` | Java missing |
| Lock file | N/A | `poetry.lock` | Java missing |
| CLI command | N/A | `poetry show --tree --no-ansi` + `poetry show --all --no-ansi` | Java missing |
| Env var overrides | N/A | `TRUSTIFY_DA_POETRY_SHOW_TREE`, `TRUSTIFY_DA_POETRY_SHOW_ALL` (base64) | Java missing |
| Workspace support | N/A | Explicitly disabled (per python-poetry/poetry#2270) | Java missing |

### JS Poetry Implementation Details

- **Detection**: `pyproject.toml` filename + `poetry.lock` exists in manifest directory
- **Version resolution**: `poetry show --all` provides resolved versions for all packages
- **Tree building**: `poetry show --tree` provides the dependency tree with indentation/box-drawing characters
- **Tree parsing**: Parses indentation depth from UTF-8 box-drawing prefixes to reconstruct parent-child relationships
- **Root component**: Reads `project.name` or `tool.poetry.name` from pyproject.toml
- **Lock file location**: Same directory as manifest (no walk-up, since Poetry has no workspace support)
- **Test fixtures**: `poetry_lock/`, `poetry_only_deps/` under `test/providers/tst_manifests/pyproject/`

---

## 4. uv

### Support Status

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Support level | **NOT SUPPORTED** -- no code at all | **FULLY SUPPORTED** | **CRITICAL GAP** |
| Provider class | N/A | `Python_uv extends Base_pyproject` | Java missing |
| Lock file | N/A | `uv.lock` | Java missing |
| CLI command | N/A | `uv export --format requirements.txt --frozen --no-hashes` | Java missing |
| Env var override | N/A | `TRUSTIFY_DA_UV_EXPORT` (base64) | Java missing |
| Workspace support | N/A | **YES** -- walk-up to find `uv.lock`, workspace root detection via `[tool.uv.workspace]` | Java missing |

### JS uv Implementation Details

- **Detection**: `pyproject.toml` filename + `uv.lock` found (with directory walk-up)
- **Resolution**: `uv export --format requirements.txt --frozen --no-hashes` produces a requirements.txt-like output with `# via` comments
- **Parsing strategy**: Uses tree-sitter (`tree-sitter-requirements.wasm`) for package/version extraction, string parsing for `# via` parent comments
- **Graph construction**: Inverts parent `# via` comments into forward dependency edges
- **Editable installs**: Handles `-e path/to/pkg` entries by reading the member's `pyproject.toml` for name/version
- **Workspace root detection**: Checks for `[tool.uv.workspace]` in `pyproject.toml` during lock file walk-up
- **Test fixtures**: `uv_lock/`, `uv_workspace/`, `pep621_ignore_and_extras/` (with `uv.lock`)

### uv Workspace Support (JS only)

The `Base_pyproject._findLockFileDir()` method walks up the directory tree looking for the lock file. When it encounters a directory with a `pyproject.toml` containing `[tool.uv.workspace]`, it stops (workspace root boundary). The `Python_uv._parseUvExport()` method also handles editable workspace members (`-e path/to/pkg`).

---

## 5. Test Coverage Comparison

### Java Client Test Fixtures

| Directory | Type | Description |
|---|---|---|
| `pip/pip_requirements_txt_no_ignore/` | requirements.txt | Basic pip with expected SBOMs |
| `pip/pip_requirements_txt_ignore/` | requirements.txt | Pip with exhortignore markers |
| `pip/pip_requirements_txt_marker_skip/` | requirements.txt | Marker-constrained deps (not installed) |
| `pip/pip_requirements_txt_marker_installed/` | requirements.txt | Marker-constrained deps (installed) |
| `pip/pip_pyproject_toml_no_ignore/` | pyproject.toml | PEP 621 with pip report |
| `pip/pip_pyproject_toml_ignore/` | pyproject.toml | PEP 621 with ignore markers |
| `pip/pip_pyproject_toml_poetry/` | pyproject.toml | Poetry deps (rejection test) |
| `pip/pip_pyproject_toml_no_metadata/` | pyproject.toml | No project name/version |
| `pip/pip_pyproject_toml_pep621_license/` | pyproject.toml | PEP 621 license field |
| `pip/pip_pyproject_toml_poetry_license/` | pyproject.toml | Poetry license field |
| `pip/pip-freeze-all.txt` | env data | Mock `pip freeze --all` output |
| `pip/pip-show.txt` | env data | Mock `pip show` output |
| `pip/pipdeptree.json` | env data | Mock `pipdeptree --json` output |

### JS Client Test Fixtures

| Directory | Type | Description |
|---|---|---|
| `pip/pip_requirements_txt_no_ignore/` | requirements.txt | Basic pip |
| `pip/pip_requirements_txt_ignore/` | requirements.txt | Pip with exhortignore |
| `pip/pip_requirements_virtual_env_txt_no_ignore/` | requirements.txt | Virtual env test |
| `pip/pip_requirements_virtual_env_with_ignore/` | requirements.txt | Virtual env + ignore |
| `pyproject/pip_pep621/` | pyproject.toml | PEP 621 + pip report |
| `pyproject/pip_pep621_ignore/` | pyproject.toml | PEP 621 + ignore |
| `pyproject/poetry_lock/` | pyproject.toml | Poetry with lock file |
| `pyproject/poetry_only_deps/` | pyproject.toml | Poetry deps only |
| `pyproject/uv_lock/` | pyproject.toml | uv with lock file |
| `pyproject/uv_workspace/` | pyproject.toml | uv workspace/monorepo |
| `pyproject/pep621_ignore_and_extras/` | pyproject.toml | Ignore + extras + uv.lock |

### Java Client Test Classes

| Class | Focus |
|---|---|
| `Python_Provider_Test` | PythonPipProvider (requirements.txt): stack, component, pipdeptree, ignore, markers |
| `Python_Pyproject_Provider_Test` | PythonPyprojectProvider: pip report parsing, poetry rejection, metadata, canonicalization |
| `PythonControllerBaseTest` | Low-level pip show/freeze parsing utilities |
| `PythonControllerRealEnvTest` | Real environment setup |
| `PythonControllerVirtualEnvTest` | Virtual environment lifecycle |

### JS Client Test Files

| File | Focus |
|---|---|
| `python_pip.test.js` | python_pip provider: real env, virtual env, component, stack, ignore |
| `python_pyproject.test.js` | All pyproject providers: pip/poetry/uv, lock file validation, workspace, ignore |

---

## 6. Summary of Parity Gaps

### Critical Gaps (missing functionality)

| # | Gap | Side Missing | Impact |
|---|---|---|---|
| 1 | **uv support** | Java | Users of uv (fast-growing tool) cannot use the Java client with pyproject.toml + uv.lock |
| 2 | **Poetry support** | Java | Java client throws an exception for Poetry projects; JS client handles them seamlessly |
| 3 | **Workspace/monorepo support** | Java | Java has no lock file walk-up or workspace detection for any Python tool |

### Moderate Gaps (behavioral differences)

| # | Gap | Details | Impact |
|---|---|---|---|
| 4 | **`trustify-da-ignore` marker in requirements.txt (JS)** | JS tree-sitter query in `requirements_parser.js` only matches `exhortignore`, not `trustify-da-ignore` | Users who use the new marker in requirements.txt will not see their ignore annotations respected in the JS client |
| 5 | **Environment marker handling** | Java strips markers at `;` and skips uninstalled marker-constrained deps; JS does not handle markers in requirements.txt parsing | JS may fail or include platform-specific deps that aren't installed |
| 6 | **Poetry rejection in pyproject.toml** | Java throws `IllegalStateException` for `[tool.poetry.dependencies]`; JS falls through to pip report (or matched by Poetry provider) | Different error behavior for same manifest when no lock file present |

### Minor Gaps (implementation differences, same outcome)

| # | Gap | Details |
|---|---|---|
| 7 | **requirements.txt parsing** | Java uses regex/string manipulation; JS uses tree-sitter WASM grammar |
| 8 | **Poetry license in pyproject.toml** | JS `Base_pyproject.readLicenseFromManifest()` checks `tool.poetry.license` as fallback; Java only checks PEP 621 fields |
| 9 | **Virtual env tmp directory** | Java: `/tmp/trustify_da_env`; JS: `/tmp/trustify_da_env_js` |

---

## 7. Architectural Notes

### Provider Matching Strategy

**Java**: Single-dispatch switch statement in `Ecosystem.resolveProvider()`. Each filename maps to exactly one provider. There is no chain/fallback mechanism. `pyproject.toml` always maps to `PythonPyprojectProvider` (pip-based).

**JS**: Provider array with priority ordering. For `pyproject.toml`, the chain is:
1. `Python_poetry` -- validates by checking for `poetry.lock`
2. `Python_uv` -- validates by checking for `uv.lock` (with walk-up)
3. `Python_pip_pyproject` -- always validates (fallback)

This architectural difference is the root cause of the uv/Poetry gap. The Java client would need either a similar provider chain or a single `pyproject.toml` provider that internally dispatches based on lock file presence.

### Shared Base Class Pattern

**JS**: `Base_pyproject` provides shared functionality for all three pyproject.toml providers (pip, poetry, uv): SBOM construction, ignore handling, canonicalization, root component extraction, and lock file walk-up.

**Java**: `PythonProvider` is the shared base, but only `PythonPipProvider` and `PythonPyprojectProvider` extend it. There is no poetry or uv subclass.

---

## 8. Recommendations

1. **Add uv support to Java client**: Create a `PythonUvProvider` that runs `uv export --format requirements.txt --frozen --no-hashes` and parses the output. The Java client will need requirements.txt parsing with `# via` comment interpretation.

2. **Add Poetry support to Java client**: Create a `PythonPoetryProvider` that runs `poetry show --tree` and `poetry show --all`, then parses the tree-drawing output for dependency structure.

3. **Implement provider chain in Java**: Modify `Ecosystem.resolveProvider()` to support multiple providers for the same manifest filename, with lock file validation as the selection mechanism (matching the JS pattern).

4. **Fix JS `trustify-da-ignore` in requirements.txt**: Update `requirements_parser.js` `getIgnoreQuery()` to match both `exhortignore` and `trustify-da-ignore` patterns.

5. **Add marker handling to JS requirements.txt**: Consider adding environment marker awareness to the JS `Python_controller` so that platform-specific deps are handled consistently.

---

## 9. Workspace/Monorepo Support -- Cross-Ecosystem Analysis

This section covers workspace/monorepo support for every ecosystem in both the Java and JavaScript clients. The analysis covers three levels of workspace support:

1. **Lock file walk-up**: When analyzing a workspace member's manifest, can the provider find the lock file at the workspace root?
2. **Workspace-aware SBOM generation**: Does the provider understand workspace structure (members, shared dependencies)?
3. **Batch/multi-manifest analysis**: Can the client discover and analyze all workspace members in a single operation?

### Shared Infrastructure

#### JS Client: `src/workspace.js`

The JS client has a dedicated shared module (`src/workspace.js`) that provides workspace discovery for batch analysis:

- **`discoverWorkspacePackages(root, opts)`** -- Discovers JS workspace members. Checks for `pnpm-workspace.yaml` first, then `package.json` `workspaces` field. Uses `fast-glob` to expand workspace glob patterns and find member `package.json` files.
- **`discoverWorkspaceCrates(root, opts)`** -- Discovers Cargo workspace members. Runs `cargo metadata --format-version 1 --no-deps` and extracts `workspace_members` + their `manifest_path` values.
- **`validatePackageJson(path)`** -- Validates that a `package.json` has required `name` and `version` fields.
- **`resolveWorkspaceDiscoveryIgnore(opts)`** -- Merges default ignore patterns (`**/node_modules/**`, `**/.git/**`) with user-provided patterns from `TRUSTIFY_DA_WORKSPACE_DISCOVERY_IGNORE` env var or `opts.workspaceDiscoveryIgnore`.
- **`filterManifestPathsByDiscoveryIgnore(paths, root, patterns)`** -- Filters discovered manifest paths against ignore patterns using `micromatch`.

The JS client also has `stackAnalysisBatch(workspaceRoot, html, opts)` in `src/index.js` that:
1. Calls `detectWorkspaceManifests(root, opts)` to auto-detect ecosystem and discover manifests
2. Sets `TRUSTIFY_DA_TRUSTIFY_WS_DIR` in opts so providers know the workspace root
3. Generates SBOMs for each member in parallel with configurable concurrency
4. Submits all SBOMs in a single batch request to the backend

#### JS Client: `TRUSTIFY_DA_TRUSTIFY_WS_DIR` Environment Variable

All JS providers that support workspace walk-up also support `TRUSTIFY_DA_TRUSTIFY_WS_DIR` (via env var or `opts`). When set, it overrides the walk-up logic and checks only the specified directory for the lock file. This is used by `stackAnalysisBatch` to ensure consistent workspace root resolution.

#### Java Client: No Shared Workspace Infrastructure

The Java client has **no equivalent** to `workspace.js`. There is no workspace discovery module, no `TRUSTIFY_DA_TRUSTIFY_WS_DIR` support, and no batch/multi-manifest analysis for workspaces. The only batch analysis in the Java client (`performBatchAnalysis`) is for container images, not workspace manifests.

---

### 9.1 npm Workspaces

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Lock file walk-up | **NO** -- `validateLockFile()` checks only `lockFileDir.resolve(lockFileName())` | **YES** -- `_findLockFileDir()` walks up directory tree | **GAP in Java** |
| Workspace root detection | **NO** | **YES** -- `_isWorkspaceRoot()` checks `package.json` `workspaces` field and `pnpm-workspace.yaml` | **GAP in Java** |
| Workspace boundary | N/A | Stops walk-up at workspace root (returns null if lock file not found there) | **GAP in Java** |
| `TRUSTIFY_DA_TRUSTIFY_WS_DIR` | **NO** | **YES** -- overrides walk-up, checks only specified directory | **GAP in Java** |
| Workspace member SBOM | N/A (would fail with "Lock file does not exist") | Runs `npm ls` from workspace root (where lock file is), generates SBOM for member's deps | **GAP in Java** |
| Workspace member discovery | **NO** | **YES** -- `discoverWorkspacePackages()` reads `package.json` `workspaces` field, expands globs | **GAP in Java** |
| Test fixtures | None for workspaces | `npm/workspace_member_with_lock/`, `npm/workspace_member_without_lock/` + dedicated tests | **GAP in Java** |

**JS Implementation Detail**: When a workspace member `package.json` is analyzed:
1. `_findLockFileDir()` walks up from the member's directory
2. It checks each directory for `package-lock.json`
3. It also checks for workspace root markers (`package.json` with `workspaces` field, `pnpm-workspace.yaml`)
4. If it hits a workspace root without a lock file, it returns `null` (fails)
5. If it finds the lock file, it returns that directory
6. `_buildDependencyTree()` runs `npm ls` from the lock file directory (workspace root)
7. The SBOM is generated for the specific member's dependencies

**Java Implementation**: `JavaScriptProviderFactory.create()` iterates over known lock file names and checks only `manifestDir.resolve(lockFileName)`. No walk-up, no workspace detection. A workspace member package.json without a local lock file will throw `IllegalStateException`.

---

### 9.2 pnpm Workspaces

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Lock file walk-up | **NO** | **YES** -- same `_findLockFileDir()` mechanism as npm, walks up for `pnpm-lock.yaml` | **GAP in Java** |
| `pnpm-workspace.yaml` detection | **NO** | **YES** -- `_isWorkspaceRoot()` checks for `pnpm-workspace.yaml` as workspace boundary | **GAP in Java** |
| Workspace boundary | N/A | `pnpm-workspace.yaml` presence stops walk-up (even if no lock file found there) | **GAP in Java** |
| `TRUSTIFY_DA_TRUSTIFY_WS_DIR` | **NO** | **YES** | **GAP in Java** |
| Workspace member discovery | **NO** | **YES** -- `discoverFromPnpmWorkspace()` parses `pnpm-workspace.yaml` `packages:` list, expands globs | **GAP in Java** |
| Test fixtures | None for workspaces | `pnpm/workspace_member_without_lock/` with `pnpm-workspace.yaml` + dedicated test | **GAP in Java** |

**JS Implementation Detail**: pnpm workspace discovery (`discoverFromPnpmWorkspace`) reads `pnpm-workspace.yaml`, parses its `packages:` YAML array, converts entries to glob patterns (e.g., `packages/*/package.json`), and uses `fast-glob` to find matching manifests.

**pnpm list command note**: The pnpm provider overrides `_buildDependencyTree()` to handle pnpm's array output format -- `pnpm ls --json` returns an array (one entry per workspace member visible), and the override extracts `tree[0]`.

---

### 9.3 Yarn Workspaces (Classic + Berry)

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Lock file walk-up | **NO** | **YES** -- same `_findLockFileDir()` mechanism, walks up for `yarn.lock` | **GAP in Java** |
| Workspace root detection | **NO** | **YES** -- same `_isWorkspaceRoot()` (checks `package.json` `workspaces` field) | **GAP in Java** |
| `TRUSTIFY_DA_TRUSTIFY_WS_DIR` | **NO** | **YES** | **GAP in Java** |
| Yarn Berry `@workspace:.` root | **YES** -- `isRoot()` checks `name.endsWith("@workspace:.")` | **YES** -- identical logic in `yarn_berry_processor.js` | **MATCH** |
| Workspace member discovery | **NO** | **YES** -- uses `discoverWorkspacePackages()` (same as npm) | **GAP in Java** |

**Both clients** correctly handle Yarn Berry's `@workspace:.` locator notation when processing the dependency tree from `yarn info --all --json`. The root package is identified by the `@workspace:.` suffix. This is a workspace-aware feature present in both clients, but only for Yarn Berry's tree parsing -- not for lock file resolution or member discovery.

---

### 9.4 Cargo Workspaces

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Lock file walk-up | **YES** -- `findOutermostCargoTomlDirectory()` walks up for `Cargo.toml` (not `Cargo.lock` directly) | **YES** -- `validateLockFile()` walks up for `Cargo.lock`, stops at `[workspace]` boundary | **MATCH** (different strategy, same result) |
| Workspace layout detection | **YES** -- `getProjectLayout()` returns `SINGLE_CRATE`, `WORKSPACE_VIRTUAL`, `WORKSPACE_WITH_ROOT_CRATE` | **YES** -- `detectCrateType()` returns identical enum values | **MATCH** |
| Virtual workspace handling | **YES** -- `handleVirtualWorkspace()`: STACK walks all members + transitive; COMPONENT reads `[workspace.dependencies]` | **YES** -- `handleVirtualWorkspace()`: identical logic | **MATCH** |
| Workspace-with-root handling | **YES** -- processes root crate deps; members included only if they appear as deps of root | **YES** -- `handleSingleCrate()` used for root crate (members naturally included via dep graph) | **MATCH** |
| `cargo metadata` invocation | **YES** -- `executeCargoMetadata()` | **YES** -- `executeCargoMetadata()` | **MATCH** |
| `workspace_members` processing | **YES** -- iterates `metadata.workspaceMembers()`, processes each member | **YES** -- iterates `metadata.workspace_members` | **MATCH** |
| `[workspace.dependencies]` reading | **YES** -- `processWorkspaceDependencies()` reads from TOML `workspace.dependencies` table | **YES** -- `getWorkspaceDepsFromManifest()` reads via `parsed.workspace?.dependencies` | **MATCH** |
| Workspace version resolution | **YES** -- reads `workspace.package.version` from root `Cargo.toml`; supports `version = { workspace = true }` inheritance | **YES** -- `getWorkspaceVersion()` reads `parsed.workspace?.package?.version` | **MATCH** |
| Member ignore patterns | **YES** -- `getMemberIgnoredDeps()` reads each member's `Cargo.toml` for ignore patterns and merges with workspace-level | **YES** -- `scanManifestForIgnored()` reads member manifests | **MATCH** |
| Path dependency PURLs | **YES** -- `createCargoPurl()` with `isPathDependency` flag adds `repository_url=local` qualifier | **YES** -- `toPathDepPurl()` adds `repository_url: 'local'` qualifier | **MATCH** |
| `TRUSTIFY_DA_TRUSTIFY_WS_DIR` | **NO** | **YES** -- overrides walk-up in `validateLockFile()` | **GAP in Java** |
| Workspace member discovery | **NO** (no batch analysis for manifests) | **YES** -- `discoverWorkspaceCrates()` runs `cargo metadata --no-deps` and extracts member manifest paths | **GAP in Java** |
| Test fixtures | `CargoProviderLockFileValidationTest` (in-memory `@TempDir`), `CargoProviderCargoParsingTest` (workspace TOML tests) | `cargo_virtual_workspace/`, `cargo_virtual_workspace_with_*` (7 fixtures), `cargo_workspace_with_root*` (3 fixtures), `workspace_member_with_lock/`, `workspace_member_without_lock/` | **Java has fewer but adequate** |

**Cargo is the most mature workspace ecosystem in both clients.** Both clients have essentially identical workspace handling for SBOM generation. The Java client's `findOutermostCargoTomlDirectory()` walks up looking for parent `Cargo.toml` files (finding the outermost one), while the JS client walks up looking for `Cargo.lock` and stops at `[workspace]` boundaries. The end result is functionally equivalent: both find the workspace root and use `cargo metadata` from there.

**Key difference**: The Java client walks up looking for the **outermost** `Cargo.toml` (going all the way to the filesystem root), while the JS client walks up looking for `Cargo.lock` and stops at `[workspace]` boundaries. The Java approach could theoretically overshoot in deeply nested directory structures, but in practice Cargo projects don't have nested workspaces.

---

### 9.5 Maven Multi-Module Projects

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Multi-module POM support | **YES** -- `provideStack()` runs `mvn dependency:tree` which inherently handles multi-module projects | **YES** -- `#getRootFromPom()` handles `projects.project` (effective POM with multiple modules) | **MATCH** |
| Effective POM parsing | **YES** -- runs `mvn help:effective-pom` and parses XML | **YES** -- `createSbomFileFromTextFormat` runs same commands | **MATCH** |
| Aggregator POM detection | **YES** -- checks for `packaging=pom` in XML parsing | **YES** -- `#getRootFromPom()` selects the project with `packaging === 'pom'` from `projects.project` | **MATCH** |
| Multi-module dependency tree | Delegated to Maven CLI (`mvn dependency:tree`) | Delegated to Maven CLI (`mvn dependency:tree`) | **MATCH** |
| Lock file concept | N/A (Maven has no lock file) | N/A (`validateLockFile()` returns `true`) | **MATCH** |
| Workspace discovery | **NO** | **NO** | **MATCH** (both missing) |
| `TRUSTIFY_DA_TRUSTIFY_WS_DIR` | **NO** | **NO** | **MATCH** |

**Note**: Maven multi-module support is inherently handled by the Maven CLI. Both clients delegate to `mvn help:effective-pom` (which produces a combined XML for all modules) and `mvn dependency:tree` (which produces a combined tree). Neither client has explicit workspace/monorepo discovery for Maven -- it's all handled by Maven's built-in parent/module resolution.

---

### 9.6 Gradle Multi-Project Builds

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| Multi-project support | Delegated to Gradle CLI (`gradle dependencies`) | Delegated to Gradle CLI (`gradle dependencies`) | **MATCH** |
| `settings.gradle` handling | Not explicitly parsed; relies on Gradle CLI to resolve project structure | Not explicitly parsed; relies on Gradle CLI | **MATCH** |
| Lock file concept | N/A (Gradle has no standard lock file) | N/A (`validateLockFile()` returns `true`) | **MATCH** |
| Workspace discovery | **NO** | **NO** | **MATCH** (both missing) |
| `TRUSTIFY_DA_TRUSTIFY_WS_DIR` | **NO** | **NO** | **MATCH** |

**Note**: Like Maven, Gradle multi-project builds are inherently handled by the Gradle CLI. Both clients run `gradle dependencies` from the manifest directory, and Gradle's own resolution handles `settings.gradle`/`settings.gradle.kts` project inclusion. Neither client has explicit workspace discovery for Gradle.

---

### 9.7 Go Workspaces (`go.work`)

| Aspect | Java Client | JS Client | Parity |
|---|---|---|---|
| `go.work` support | **NO** -- no mention of `go.work` anywhere in code | **NO** -- no mention of `go.work` or workspace in Go provider | **MATCH** (both missing) |
| Lock file concept | N/A (`go.sum` not validated) | N/A (`validateLockFile()` returns `true`) | **MATCH** |
| Workspace discovery | **NO** | **NO** | **MATCH** |
| `TRUSTIFY_DA_TRUSTIFY_WS_DIR` | **NO** | **NO** | **MATCH** |

**Note**: Go 1.18+ supports workspaces via `go.work` files. Neither client has any awareness of Go workspaces. Both clients only support individual `go.mod` files and delegate dependency resolution to `go mod graph` and `go mod edit -json`. This means Go workspace projects must be analyzed module by module.

---

### 9.8 Python (pip/Poetry/uv)

Covered in detail in Sections 1--4 above. Summary:

| Sub-ecosystem | Java Client | JS Client | Parity |
|---|---|---|---|
| pip (requirements.txt) | No workspace concept | No workspace concept | **MATCH** |
| pip (pyproject.toml) | No workspace concept | No workspace concept | **MATCH** |
| Poetry | REJECTED | No workspace (by design -- Poetry doesn't support workspaces per python-poetry/poetry#2270) | N/A |
| uv | NOT SUPPORTED | **YES** -- lock file walk-up, `[tool.uv.workspace]` detection, editable member handling | **GAP in Java** |

---

### 9.9 Cross-Ecosystem Workspace Summary Table

| Ecosystem | Feature | Java | JS | Parity |
|---|---|---|---|---|
| **npm** | Lock file walk-up | NO | YES | **GAP** |
| **npm** | Workspace root detection (`workspaces` field) | NO | YES | **GAP** |
| **npm** | `TRUSTIFY_DA_TRUSTIFY_WS_DIR` | NO | YES | **GAP** |
| **npm** | Workspace member discovery | NO | YES | **GAP** |
| **pnpm** | Lock file walk-up | NO | YES | **GAP** |
| **pnpm** | `pnpm-workspace.yaml` detection | NO | YES | **GAP** |
| **pnpm** | `TRUSTIFY_DA_TRUSTIFY_WS_DIR` | NO | YES | **GAP** |
| **pnpm** | Workspace member discovery | NO | YES | **GAP** |
| **yarn** | Lock file walk-up | NO | YES | **GAP** |
| **yarn** | Berry `@workspace:.` root | YES | YES | MATCH |
| **yarn** | `TRUSTIFY_DA_TRUSTIFY_WS_DIR` | NO | YES | **GAP** |
| **yarn** | Workspace member discovery | NO | YES | **GAP** |
| **Cargo** | Lock file walk-up | YES | YES | MATCH |
| **Cargo** | Workspace layout detection | YES | YES | MATCH |
| **Cargo** | Virtual workspace handling | YES | YES | MATCH |
| **Cargo** | `TRUSTIFY_DA_TRUSTIFY_WS_DIR` | NO | YES | **GAP** |
| **Cargo** | Workspace member discovery | NO | YES | **GAP** |
| **Maven** | Multi-module (via Maven CLI) | YES | YES | MATCH |
| **Maven** | Workspace discovery | NO | NO | MATCH |
| **Gradle** | Multi-project (via Gradle CLI) | YES | YES | MATCH |
| **Gradle** | Workspace discovery | NO | NO | MATCH |
| **Go** | `go.work` support | NO | NO | MATCH |
| **Go** | Workspace discovery | NO | NO | MATCH |
| **uv** | Lock file walk-up | N/A | YES | **GAP** |
| **uv** | `[tool.uv.workspace]` detection | N/A | YES | **GAP** |
| **Poetry** | Workspace support | N/A | Disabled by design | N/A |
| **pip** | Workspace support | NO | NO | MATCH |

---

### 9.10 Workspace Recommendations

1. **Add lock file walk-up for JS ecosystems in Java client**: The Java `JavaScriptProvider.validateLockFile()` and `JavaScriptProviderFactory.create()` need directory walk-up to find lock files at workspace roots. The JS client's `Base_javascript._findLockFileDir()` is a good reference implementation. This is the **highest priority** workspace parity gap because npm/pnpm/yarn workspaces are very common in JavaScript projects, and the Java client (used by the IntelliJ plugin) will fail outright on workspace member `package.json` files.

2. **Add `TRUSTIFY_DA_TRUSTIFY_WS_DIR` support to Java client**: All providers should accept this option to allow explicit workspace root specification, bypassing walk-up logic. This is particularly important for IDE integrations where the workspace root is known.

3. **Add workspace member discovery to Java client**: Implement equivalents of `discoverWorkspacePackages()` and `discoverWorkspaceCrates()` to enable batch analysis of workspace members. This requires a new API entry point (the Java client currently only has batch for images).

4. **Add `go.work` support to both clients**: Go workspaces are increasingly common. Both clients should detect `go.work` files and use `go work sync` / `go mod graph` appropriately. This is a parity match (both missing) but a feature gap for both.

5. **Consider workspace root detection in Java `Ecosystem.getProvider()`**: Currently `getProvider()` calls `provider.validateLockFile(manifestPath.getParent())` -- for workspace members, the lock file is not in the parent directory. The Java client needs either a walk-up in `validateLockFile` or a pre-resolution step that finds the workspace root first.
