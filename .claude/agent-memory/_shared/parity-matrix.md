# Feature Parity Matrix -- Dependency Analytics Clients

Last updated: 2026-04-23 (PR#428 refactored to `uv export`; PR#421 Go MVS fix merged; PR#423 npm null-version fix merged; TC-3818 partially resolved — 2 of 4 sub-issues fixed)

## Ecosystem Support

| Ecosystem | Sub-ecosystem | Manifest | Java Client | JS Client | Status |
|---|---|---|---|---|---|
| Maven | - | pom.xml | JavaMavenProvider | java_maven.js | **Parity (gaps in both)** |
| Gradle | Groovy | build.gradle | GradleProvider | java_gradle_groovy.js | **Parity (gaps in both)** |
| Gradle | Kotlin | build.gradle.kts | GradleProvider | java_gradle_kotlin.js | **Parity (gaps in both)** |
| npm | - | package.json | JavaScriptNpmProvider | javascript_npm.js | **Parity** (Java null-version fix landed PR#423) |
| pnpm | - | package.json | JavaScriptPnpmProvider | javascript_pnpm.js | **Parity** |
| yarn | Classic | package.json | JavaScriptYarnProvider | javascript_yarn.js | **Parity** |
| yarn | Berry | package.json | JavaScriptYarnProvider | javascript_yarn.js | **Parity** |
| Go | modules | go.mod | GoModulesProvider | golang_gomodules.js | **Functional differences** (Java MVS fix landed PR#421; JS `toolchain@` filter still missing) |
| **Python** | **pip (requirements.txt)** | **requirements.txt** | **PythonPipProvider** | **python_pip.js** | **Parity (minor gaps -- see detail)** |
| **Python** | **pip (pyproject.toml)** | **pyproject.toml** | **PythonPyprojectProvider** | **Python_pip_pyproject** | **Parity (aligned after JS TC-4065 + Java #406; minor gaps)** |
| **Python** | **Poetry** | **pyproject.toml** | **REJECTED** | **Python_poetry** | **CRITICAL GAP -- Java missing** |
| **Python** | **uv** | **pyproject.toml** | **PythonUvProvider (PR#428 open)** | **Python_uv** | **IN PROGRESS -- Java PR#428 now uses `uv export` (aligned); remaining gaps: no lock walk-up, no workspace support, no editable member handling** |
| Cargo | - | Cargo.toml | CargoProvider | rust_cargo.js | **BEST PARITY** |

---

## Per-Ecosystem Feature Detail

### Maven (pom.xml)

| Feature | Java | JS | Parity |
|---|---|---|---|
| CLI commands (tree + effective POM) | Yes | Yes | MATCH |
| Wrapper support (mvnw) | Yes | Yes | MATCH |
| Root from effective POM | Yes | Yes | MATCH |
| Multi-module (aggregator POM) | Yes | Yes | MATCH |
| `exhortignore` in XML comments | Yes | Yes | MATCH |
| `trustify-da-ignore` in XML comments | Yes | **NO** | **GAP in JS** |
| `TRUSTIFY_DA_MVN_USER_SETTINGS` | Yes | **NO** | **GAP in JS** |
| `TRUSTIFY_DA_MVN_LOCAL_REPO` | Yes | **NO** | **GAP in JS** |
| `TRUSTIFY_DA_MVN_ARGS` (extra args) | **NO** | Yes | **GAP in Java** |
| JAVA_HOME passthrough | Yes (explicit) | No (inherits) | Minor |
| License from pom.xml | Yes | Yes | MATCH |
| Test dependency filtering | Yes | Yes | MATCH |

### Gradle (build.gradle / build.gradle.kts)

| Feature | Java | JS | Parity |
|---|---|---|---|
| `gradle dependencies` + `gradle properties` | Yes | Yes | MATCH |
| Wrapper support (gradlew) | **NO** | Yes | **GAP in Java** |
| Groovy + Kotlin variants | Single class | Subclasses with regex variants | MATCH (behavior) |
| runtimeClasspath + compileClasspath extraction | Yes | Yes | MATCH |
| REQUIRED/OPTIONAL scope | Yes | Yes | MATCH |
| `exhortignore` in comments | Yes | Yes | MATCH |
| `trustify-da-ignore` in comments | Yes | **NO** | **GAP in JS** |
| libs.versions.toml catalog resolution | Yes | Yes | MATCH |
| Root from `gradle properties` | Yes | Yes | MATCH |
| Ignored dep filtering | Coordinate-based | Version-aware | Minor diff |

### npm (package.json + package-lock.json)

| Feature | Java | JS | Parity |
|---|---|---|---|
| `npm ls --json` tree parsing | Yes | Yes | MATCH |
| Lock file walk-up (workspace) | Yes | Yes | MATCH |
| Workspace root detection | Yes | Yes | MATCH |
| `TRUSTIFY_DA_TRUSTIFY_WS_DIR` | Yes | Yes | MATCH |
| `exhortignore` in package.json | Yes | Yes | MATCH |
| `trustify-da-ignore` in package.json | Yes | **NO** | **GAP in JS** |
| Peer/optional dep handling | Yes | Yes | MATCH |
| Version null check at root | Passes `null` version to PURL, processes subtree unconditionally | No check — processes versionless entries and recurses | **FIXED** (PR#423, commit `cddf65b`): Java no longer skips subtree for null-version entries |
| `filterIgnoredDeps` behavior | Recursive removal: ignored dep + all transitive-only children (insensitive mode) | Non-recursive: removes only the ignored dep, orphaned transitives remain | **GAP in JS**: JS over-counts when `exhortignore` is used (still unfixed) |
| Scoped package PURLs | Yes | Yes | MATCH |
| fnm/nvm integration | **NO** | Yes | **GAP in Java** |
| `node_modules/.bin` auto-detect | **NO** | Yes | **GAP in Java** |

### pnpm (package.json + pnpm-lock.yaml)

| Feature | Java | JS | Parity |
|---|---|---|---|
| `pnpm ls --json` array handling | Yes (`depTree.get(0)`) | Yes (`tree[0]`) | MATCH |
| Lock file walk-up (workspace) | Yes | Yes | MATCH |
| `pnpm-workspace.yaml` detection | Yes | Yes | MATCH |
| `exhortignore` in package.json | Yes | Yes | MATCH |
| `trustify-da-ignore` in package.json | Yes | **NO** | **GAP in JS** |
| Lock file update command | `pnpm i --lockfile-only` | `pnpm install --frozen-lockfile` | **DIFFERENT** semantics |

### yarn (Classic + Berry)

| Feature | Java | JS | Parity |
|---|---|---|---|
| Version detection (Classic vs Berry) | Yes (`yarn -v`) | Yes (`yarn --version`) | MATCH |
| Classic: `yarn list --json` | Yes | Yes | MATCH |
| Berry: `yarn info --json` (NDJSON) | Yes | Yes | MATCH |
| Berry: `@workspace:.` root detection | Yes | Yes | MATCH |
| Berry: locator/virtual locator parsing | Yes | Yes | MATCH |
| Lock file walk-up (workspace) | Yes | Yes | MATCH |
| `exhortignore` in package.json | Yes | Yes | MATCH |
| `trustify-da-ignore` in package.json | Yes | **NO** | **GAP in JS** |

### Go modules (go.mod)

| Feature | Java | JS | Parity |
|---|---|---|---|
| `go mod graph` parsing | Yes | Yes | MATCH |
| `go mod edit -json` (root + direct deps) | **NO** | Yes | **GAP in Java** |
| Direct dep classification | All root edges = direct | Uses `Indirect` flag from `go mod edit -json` | **DIFFERENT** (JS more correct) |
| Root module version from git | Yes (VCS tag/pseudo-version) | **NO** (hardcodes `v0.0.0`) | **GAP in JS** |
| MVS version resolution | Yes (uses `Map.merge()` to combine children on key collision) | Yes | **FIXED** (PR#421, commit `c7f761d`): Java now merges children instead of overwriting |
| `TRUSTIFY_DA_GO_MVS_LOGIC_ENABLED` | Yes | Yes | MATCH |
| `MATCH_MANIFEST_VERSIONS` (default false) | Yes | Yes | MATCH |
| `exhortignore` in go.mod comments | Yes | Yes | MATCH |
| `trustify-da-ignore` in go.mod comments | Yes | **NO** | **GAP in JS** |
| go.mod parsing | Manual regex/string | tree-sitter WASM | Different mechanism |
| Toolchain entry filtering | `isGoToolchainEntry()` filters both `go@` and `toolchain@` | Only filters `go@` (missing `toolchain@`) | **GAP in JS** (still unfixed) |

### Cargo (Cargo.toml + Cargo.lock) -- BEST PARITY

| Feature | Java | JS | Parity |
|---|---|---|---|
| `cargo metadata --format-version 1` | Yes | Yes | MATCH |
| Lock file walk-up | Yes (outermost Cargo.toml) | Yes ([workspace] boundary) | MATCH |
| Crate type detection (single/virtual/root) | Yes | Yes | MATCH |
| Virtual workspace handling | Yes | Yes | MATCH |
| Workspace-with-root handling | Yes | Yes | MATCH |
| `[workspace.dependencies]` reading | Yes | Yes | MATCH |
| Workspace version inheritance | Yes | Yes | MATCH |
| Path dependency PURLs (`repository_url=local`) | Yes | Yes | MATCH |
| `exhortignore` in Cargo.toml | Yes | Yes | MATCH |
| `trustify-da-ignore` in Cargo.toml | Yes | Yes | **MATCH** |
| Member ignore pattern merging | Yes | Yes | MATCH |
| Runtime dep filtering (skip dev/build) | Yes | Yes | MATCH |
| `TRUSTIFY_DA_TRUSTIFY_WS_DIR` | **NO** | Yes | **GAP in Java** |
| Workspace member discovery (batch) | Yes | Yes | MATCH |

### Python (requirements.txt + pyproject.toml)

#### pip (requirements.txt)

| Feature | Java | JS | Parity |
|---|---|---|---|
| pip freeze + pip show resolution | Yes | Yes | MATCH |
| pipdeptree alternative (`TRUSTIFY_DA_PIP_USE_DEP_TREE`) | Yes | Yes | MATCH |
| Virtual env support | Yes (PythonControllerVirtualEnv) | Yes (Python_controller) | MATCH |
| Real env support | Yes (PythonControllerRealEnv) | Yes (Python_controller realEnvironment) | MATCH |
| Best-effort install | Yes | Yes | MATCH |
| `MATCH_MANIFEST_VERSIONS` (default true) | Yes | Yes | MATCH |
| `exhortignore` in requirements.txt | Yes | Yes | MATCH |
| `trustify-da-ignore` in requirements.txt | Yes | **NO** (tree-sitter query hardcodes `exhortignore`) | **GAP in JS** |
| PEP 508 marker handling | Yes (strips `;`, skips uninstalled; TC-4044 done) | **NO** (throws `PackageNotInstalled` error; TC-4043 in review) | **GAP in JS** (Java fixed, JS fix pending) |
| Requirements parsing mechanism | Regex/string | tree-sitter WASM | Different (same result) |

#### pip (pyproject.toml / PEP 621)

| Feature | Java | JS | Parity |
|---|---|---|---|
| `pip install --dry-run --report` resolution | Yes (`PythonPyprojectProvider`) | Yes (`Python_pip_pyproject`, TC-4065) | **MATCH (newly aligned)** |
| `TRUSTIFY_DA_PIP_REPORT` env var (base64) | Yes | Yes | MATCH |
| Root entry detection via `dir_info` | Yes (first `dir_info` = root, rest = deps) | Yes (fixed in `2e8b33f` to match Java) | **MATCH (newly aligned)** |
| Direct dep extraction from `requires_dist` | Yes | Yes | MATCH |
| Extra marker filtering (`; extra ==`) | Yes | Yes | MATCH |
| Name canonicalization | Yes (`toLowerCase().replaceAll("[-_.]+", "-")`) | Yes (`.toLowerCase().replace(/[-_.]+/g, '-')`) | MATCH |
| Cycle detection | Yes (`visited` Set) | Yes (`visited` Set via `_collectTransitive`) | MATCH |
| `.egg-info` cleanup after `pip --dry-run` | **NO** | Yes (`_cleanupEggInfo`) | **GAP in Java** |
| Poetry rejection | Throws `IllegalStateException` | No (fallback provider) | **DIFFERENT** (by design) |
| `exhortignore` in pyproject.toml | Yes | Yes | MATCH |
| `trustify-da-ignore` in pyproject.toml | Yes | Yes | MATCH |
| Poetry-style ignore patterns | N/A (Poetry rejected) | Yes | JS only (by design) |

#### Poetry

| Feature | Java | JS | Parity |
|---|---|---|---|
| Support level | **REJECTED** (IllegalStateException) | Fully supported | **CRITICAL GAP** |
| Provider class | N/A | Python_poetry extends Base_pyproject | Java missing |
| Lock file | N/A | poetry.lock (same dir only, no walk-up) | Java missing |
| CLI commands | N/A | `poetry show --tree` + `poetry show --all` | Java missing |
| Env var overrides | N/A | `TRUSTIFY_DA_POETRY_SHOW_TREE`, `TRUSTIFY_DA_POETRY_SHOW_ALL` | Java missing |
| Workspace support | N/A | Explicitly disabled (Poetry has no workspace support) | N/A |

#### uv

| Feature | Java | JS | Parity |
|---|---|---|---|
| Support level | **PR#428 open** (`PythonUvProvider`) | Fully supported | **IN PROGRESS** |
| Provider class | `PythonUvProvider extends PythonProvider` | `Python_uv extends Base_pyproject` | MATCH (structural) |
| Lock file | `uv.lock` (manifest dir only) | `uv.lock` (with directory walk-up) | **GAP in Java** (no walk-up) |
| CLI command | `uv export --format requirements.txt --frozen --no-hashes --no-dev --no-emit-project` | `uv export --format requirements.txt --frozen --no-hashes --no-dev --no-emit-project` | **MATCH** (aligned in commit `a9174b5`) |
| Dev dependency exclusion | Yes (`--no-dev` flag) | Yes (`--no-dev` flag) | **MATCH** |
| Self-reference exclusion | Yes (`--no-emit-project` flag) | Yes (`--no-emit-project` flag) | **MATCH** |
| Env var override | `TRUSTIFY_DA_UV_EXPORT` (plain text) | `TRUSTIFY_DA_UV_EXPORT` (base64) | MATCH (name); encoding differs |
| `TRUSTIFY_DA_UV_PATH` | Yes (via `Operations.getCustomPathOrElse`) | Yes (via `getCustomPath`) | MATCH |
| Workspace root detection | **NO** | `[tool.uv.workspace]` in pyproject.toml | **GAP in Java** |
| Lock file walk-up | **NO** | Yes (Base_pyproject._findLockFileDir) | **GAP in Java** |
| `TRUSTIFY_DA_WORKSPACE_DIR` | **NO** | Yes | **GAP in Java** |
| Editable member handling (`-e path/to/pkg`) | Skips silently | Yes (_parseUvExport reads member pyproject.toml) | **GAP in Java** |
| Batch workspace member discovery | **NO** | **NO** (workspace.js only handles JS + Cargo) | MATCH (both missing) |
| Ignore markers (`exhortignore` + `trustify-da-ignore`) | Yes (via `PyprojectTomlUtils` + `IgnorePatternDetector`) | Yes (via `Base_pyproject._getIgnoredDeps` + `IGNORE_MARKERS`) | MATCH |
| Name canonicalization | `toLowerCase().replaceAll("[-_.]+", "-")` | `.toLowerCase().replace(/[-_.]+/g, '-')` | MATCH |
| Root metadata from pyproject.toml | Yes (`project.name`, `project.version`) | Yes | MATCH |

See `parity-report-uv-2026-04-22.md` for full analysis of PR#428 vs JS implementation.

---

## Cross-Cutting Feature Matrix

### Ignore Marker Support

| Ecosystem | `exhortignore` Java | `exhortignore` JS | `trustify-da-ignore` Java | `trustify-da-ignore` JS |
|---|---|---|---|---|
| Maven | YES | YES | YES | **NO** |
| Gradle | YES | YES | YES | **NO** |
| npm/pnpm/yarn | YES | YES | YES | **NO** |
| Go | YES | YES | YES | **NO** |
| Cargo | YES | YES | YES | YES |
| pip (requirements.txt) | YES | YES | YES | **NO** (tree-sitter query in `requirements_parser.js:24` hardcodes `exhortignore`) |
| pyproject.toml (all providers) | YES | YES | YES | YES (via `Base_pyproject.IGNORE_MARKERS`) |

### Wrapper Support

| Tool | Java | JS |
|---|---|---|
| mvnw (Maven) | YES | YES |
| gradlew (Gradle) | **NO** | YES |

### Workspace Support

| Ecosystem | Java Lock Walk-up | JS Lock Walk-up | Java `WORKSPACE_DIR` | JS `WORKSPACE_DIR` | Java Batch Discovery | JS Batch Discovery |
|---|---|---|---|---|---|---|
| npm | YES | YES | YES | YES | YES | YES |
| pnpm | YES | YES | YES | YES | YES | YES |
| yarn | YES | YES | YES | YES | YES | YES |
| Cargo | YES | YES | NO | YES | YES | YES |
| Maven | N/A (CLI) | N/A (CLI) | NO | NO | NO | NO |
| Gradle | N/A (CLI) | N/A (CLI) | NO | NO | NO | NO |
| Go | NO | NO | NO | NO | NO | NO |
| uv (Python) | **NO** (PR#428 uses `uv export` but no walk-up) | YES | **NO** (PR#428) | YES | NO | NO (workspace.js only handles JS + Cargo) |
| Poetry | N/A (rejected) | NO (by design) | N/A | Accepted but no walk-up | N/A | N/A |
| pip (requirements.txt) | NO | NO | NO | NO | NO | NO |
| pip (pyproject.toml) | N/A | **Dead code** (dummy lock `.pip-lock-nonexistent`) | N/A | **Accepted but silently ignored** | N/A | NO |

### MATCH_MANIFEST_VERSIONS Defaults

| Ecosystem | Java Default | JS Default | Parity |
|---|---|---|---|
| pip | `true` | `"true"` | MATCH |
| Go | `false` | `"false"` | MATCH |

---

## Provider Architecture Comparison

| Aspect | Java Client | JS Client |
|---|---|---|
| Maven/Gradle shared base | None (separate classes) | `Base_Java` (shared `parseDependencyTree`, `selectToolBinary`, `traverseForWrapper`) |
| npm/pnpm/yarn shared base | `JavaScriptProvider` (abstract) | `Base_javascript` (abstract) |
| npm/pnpm/yarn dispatch | `JavaScriptProviderFactory.create()` iterates lock files | Provider array, each with `validateLockFile()` |
| Yarn Classic/Berry | `YarnProcessor` interface -> `YarnClassicProcessor`, `YarnBerryProcessor` | Same: `Yarn_classic_processor`, `Yarn_berry_processor` |
| pyproject.toml dispatch | `PythonProviderFactory.create()`: checks `uv.lock` -> `PythonUvProvider`, else `PythonPyprojectProvider` (pip). Rejects Poetry. (PR#428) | Provider chain: Poetry -> uv -> pip fallback (priority order in `provider.js:32-34`) |
| Ignore pattern detection | Centralized `IgnorePatternDetector` utility (both markers) | Per-ecosystem, inconsistent (some have both markers, most only have `exhortignore`) |
| Binary/tool selection | `Operations.getExecutable()`, `Operations.getCustomPathOrElse()` | `getCustomPath()` from `tools.js` |
| Manifest parsing | tree-sitter for Go; XML streaming for Maven; regex for others | tree-sitter for Go, pip; XML parser lib for Maven; regex for others |

---

## Workspace Support -- Detailed Analysis

Python/uv workspace details are in the Per-Ecosystem Feature Detail > Python > uv section above.

### JS Client Internal Workspace Inconsistencies

The JS client has workspace support across several providers, but an internal consistency audit (2026-04-15) found the following issues:

| # | Issue | Severity | File | Detail |
|---|---|---|---|---|
| 1 | `Python_pip_pyproject` has dead walk-up code | Medium | `python_pip_pyproject.js:17` | `_lockFileName()` returns `'.pip-lock-nonexistent'`, so `_findLockFileDir` inherited from `Base_pyproject` always returns `null`. `TRUSTIFY_DA_WORKSPACE_DIR` is accepted but silently has no effect. |
| 2 | No uv batch discovery in `workspace.js` | Medium | `index.js:325-347` | `detectWorkspaceManifests` only handles `javascript` and `cargo`. uv workspaces can't use `stackAnalysisBatch()` despite `Base_pyproject._isWorkspaceRoot` already detecting `[tool.uv.workspace]`. |
| 3 | Provider chain priority is implicit | Low | `provider.js:23-35` | Poetry > uv > pip_pyproject ordering enforced by array position, not explicit priority. If both `poetry.lock` and `uv.lock` exist, Poetry wins (undocumented). |
| 4 | Cargo workspace detection uses regex | Low | `rust_cargo.js:131-139` | Uses `/\[workspace\]/` regex instead of TOML parsing. Other providers (Base_javascript, Base_pyproject) use structured parsing for workspace root detection. |
| 5 | Duplicate `TRUSTIFY_DA_WORKSPACE_DIR` in typedef | Trivial | `index.js:62,68` | Options typedef declares the property twice. |

---

## IDE Extension Support (not yet analyzed)

| Feature | IntelliJ | VS Code | Status |
|---|---|---|---|
| Diagnostics | TBD | TBD | Not yet analyzed |
| Quick fixes | TBD | TBD | Not yet analyzed |
| Vulnerability report | TBD | TBD | Not yet analyzed |

---

## Recommendations (Priority Order)

### P0: Critical -- Blocking functional gaps

#### 1. ~~Merge Java PR#415 (TC-3863)~~ -- RESOLVED (merged 2026-04-19)
PR#415 merged and TC-3863 closed. JS workspace discovery + batch API is now landed in the Java client. All previously noted blocking issues were resolved before merge. Remaining minor gaps (programmatic `opts.*` overrides, CLI `stack-batch` command, glob negation pattern semantics, lock file walk-up checks all types) are accepted as intentional differences.

#### 2. Add Poetry support to Java client
Java throws `IllegalStateException` for `[tool.poetry.dependencies]`. Create `PythonPoetryProvider` using `poetry show --tree` + `poetry show --all`, with `poetry.lock` validation. Reference JS implementation in `python_poetry.js`.

#### 3. ~~Align uv resolution strategy in Java client~~ -- PARTIALLY RESOLVED (commit `a9174b5`, 2026-04-23)
PR#428 now uses `uv export` matching JS. Resolution strategy, dev-dep exclusion, and self-reference exclusion are aligned. Remaining gaps: no lock file walk-up, no workspace root detection, no `TRUSTIFY_DA_WORKSPACE_DIR`, no editable member handling. See `parity-report-uv-2026-04-22.md` for detailed comparison.

#### 4. Implement pyproject.toml provider chain in Java
Java's `Ecosystem.resolveProvider()` maps `pyproject.toml` to a single `PythonPyprojectProvider`. Need a chain/priority mechanism: Poetry (if `poetry.lock` exists) -> uv (if `uv.lock` exists, with walk-up) -> pip fallback. Reference JS provider ordering in `provider.js:32-34`.

### P1: High -- Significant behavioral differences

#### 5. Fix `trustify-da-ignore` in JS client (ALL non-pyproject ecosystems)
Create a centralized ignore marker utility in JS (similar to Java's `IgnorePatternDetector`). Affected files:
- `requirements_parser.js:24` -- tree-sitter query hardcodes `exhortignore`
- Maven, Gradle, Go, npm/pnpm/yarn providers (all only check `exhortignore`)
The pyproject.toml providers already handle both markers via `Base_pyproject.IGNORE_MARKERS`.

#### 6. Add PEP 508 marker handling to JS pip provider -- PARTIALLY RESOLVED
Java side fixed: TC-4044 closed, correctly skips marker-constrained packages not installed in the environment (commit `1aee31c`). JS side fix in progress: TC-4043 is in Review. JS `bringAllDependencies()` still throws `PackageNotInstalled` error instead of silently skipping uninstalled marker-constrained deps.

#### 7. Add `.egg-info` cleanup to Java `PythonPyprojectProvider`
JS client (`Python_pip_pyproject._cleanupEggInfo`) cleans up `.egg-info` directories created as a side effect of `pip install --dry-run`. Java `getPipReportOutput()` does not. This leaves filesystem artifacts.

### P2: Medium -- Feature gaps

#### 8. Add `gradlew` wrapper support to Java client
The Java `GradleProvider` should use the same wrapper traversal pattern already implemented in `JavaMavenProvider.traverseForMvnw()`.

#### 9. Add `go mod edit -json` to Java Go provider
For correct direct vs indirect dependency classification, matching JS behavior.

#### 10. ~~Add lock file walk-up for JS ecosystems in Java client~~ -- RESOLVED (PR#415 merged 2026-04-19)
Covered by PR#415 for npm/pnpm/yarn.

#### 11. Add `TRUSTIFY_DA_MVN_USER_SETTINGS` and `TRUSTIFY_DA_MVN_LOCAL_REPO` to JS client

#### 12. Add `TRUSTIFY_DA_MVN_ARGS` to Java client

#### 13. Add Go root module version from git to JS client

### P3: Low -- Minor differences

#### 14. Unify pnpm lock file update command
Java: `pnpm i --lockfile-only` vs JS: `pnpm install --frozen-lockfile`.

#### 15. ~~Add Cargo workspace batch discovery to Java client~~ -- RESOLVED (PR#415 merged 2026-04-19)
Java `discoverWorkspaceManifests()` now checks Cargo first (Cargo.toml + Cargo.lock -> `cargo metadata --no-deps`), then JS. Matches JS client's `detectWorkspaceManifests()` pattern.

#### 16. Add uv workspace batch discovery to JS client
JS `workspace.js` currently only discovers JS and Cargo workspaces, not uv workspaces. The uv provider supports workspace lock file walk-up at the individual SBOM level, but `stackAnalysisBatch()` cannot auto-discover uv workspace members.

#### 17. Fix `Python_pip_pyproject` dead walk-up / silent `WORKSPACE_DIR` (JS client)
`python_pip_pyproject.js:17` uses dummy lock file `.pip-lock-nonexistent`, making inherited `_findLockFileDir` always return `null`. `TRUSTIFY_DA_WORKSPACE_DIR` is accepted but silently ignored. Either make it functional or document it clearly.

#### 18. Document pyproject.toml provider chain priority (JS client)
Provider chain ordering (Poetry > uv > pip_pyproject) in `provider.js:23-35` is enforced by array position only. Add a comment explaining the priority and the behavior when both `poetry.lock` and `uv.lock` exist.
