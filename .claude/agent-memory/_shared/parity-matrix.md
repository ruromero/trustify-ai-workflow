# Feature Parity Matrix -- Dependency Analytics Clients

Last updated: 2026-04-14 (full ecosystem analysis)

## Ecosystem Support

| Ecosystem | Sub-ecosystem | Manifest | Java Client | JS Client | Status |
|---|---|---|---|---|---|
| Maven | - | pom.xml | JavaMavenProvider | java_maven.js | **Parity (gaps in both)** |
| Gradle | Groovy | build.gradle | GradleProvider | java_gradle_groovy.js | **Parity (gaps in both)** |
| Gradle | Kotlin | build.gradle.kts | GradleProvider | java_gradle_kotlin.js | **Parity (gaps in both)** |
| npm | - | package.json | JavaScriptNpmProvider | javascript_npm.js | **Parity (gaps in Java: workspace) + BUG TC-3818** |
| pnpm | - | package.json | JavaScriptPnpmProvider | javascript_pnpm.js | **Parity (gaps in Java: workspace) + BUG TC-3818** |
| yarn | Classic | package.json | JavaScriptYarnProvider | javascript_yarn.js | **Parity (gaps in Java: workspace)** |
| yarn | Berry | package.json | JavaScriptYarnProvider | javascript_yarn.js | **Parity (gaps in Java: workspace)** |
| Go | modules | go.mod | GoModulesProvider | golang_gomodules.js | **Functional differences + BUG TC-3818** |
| **Python** | **pip (requirements.txt)** | **requirements.txt** | **PythonPipProvider** | **python_pip.js** | **Parity (minor gaps)** |
| **Python** | **pip (pyproject.toml)** | **pyproject.toml** | **PythonPyprojectProvider** | **Python_pip_pyproject** | **Parity (minor gaps)** |
| **Python** | **Poetry** | **pyproject.toml** | **REJECTED** | **Python_poetry** | **CRITICAL GAP -- Java missing** |
| **Python** | **uv** | **pyproject.toml** | **NOT SUPPORTED** | **Python_uv** | **CRITICAL GAP -- Java missing** |
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
| Lock file walk-up (workspace) | **NO** | Yes | **GAP in Java** |
| Workspace root detection | **NO** | Yes | **GAP in Java** |
| `TRUSTIFY_DA_TRUSTIFY_WS_DIR` | **NO** | Yes | **GAP in Java** |
| `exhortignore` in package.json | Yes | Yes | MATCH |
| `trustify-da-ignore` in package.json | Yes | **NO** | **GAP in JS** |
| Peer/optional dep handling | Yes | Yes | MATCH |
| Version null check at root | `addDependenciesFromKey()` skips entry AND entire subtree if version is null | No check — processes versionless entries and recurses | **BUG TC-3818**: Java under-counts for `file:` deps, workspaces, linked packages |
| `filterIgnoredDeps` behavior | Recursive removal: ignored dep + all transitive-only children (insensitive mode) | Non-recursive: removes only the ignored dep, orphaned transitives remain | **BUG TC-3818**: JS over-counts when `exhortignore` is used |
| Scoped package PURLs | Yes | Yes | MATCH |
| fnm/nvm integration | **NO** | Yes | **GAP in Java** |
| `node_modules/.bin` auto-detect | **NO** | Yes | **GAP in Java** |

### pnpm (package.json + pnpm-lock.yaml)

| Feature | Java | JS | Parity |
|---|---|---|---|
| `pnpm ls --json` array handling | Yes (`depTree.get(0)`) | Yes (`tree[0]`) | MATCH |
| Lock file walk-up (workspace) | **NO** | Yes | **GAP in Java** |
| `pnpm-workspace.yaml` detection | **NO** | Yes | **GAP in Java** |
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
| Lock file walk-up (workspace) | **NO** | Yes | **GAP in Java** |
| `exhortignore` in package.json | Yes | Yes | MATCH |
| `trustify-da-ignore` in package.json | Yes | **NO** | **GAP in JS** |

### Go modules (go.mod)

| Feature | Java | JS | Parity |
|---|---|---|---|
| `go mod graph` parsing | Yes | Yes | MATCH |
| `go mod edit -json` (root + direct deps) | **NO** | Yes | **GAP in Java** |
| Direct dep classification | All root edges = direct | Uses `Indirect` flag from `go mod edit -json` | **DIFFERENT** (JS more correct) |
| Root module version from git | Yes (VCS tag/pseudo-version) | **NO** (hardcodes `v0.0.0`) | **GAP in JS** |
| MVS version resolution | Yes | Yes | **BUG TC-3818**: Java `HashMap.put()` overwrites children on key collision in `getFinalPackagesVersionsForModule()` — drops reachable transitive deps |
| `TRUSTIFY_DA_GO_MVS_LOGIC_ENABLED` | Yes | Yes | MATCH |
| `MATCH_MANIFEST_VERSIONS` (default false) | Yes | Yes | MATCH |
| `exhortignore` in go.mod comments | Yes | Yes | MATCH |
| `trustify-da-ignore` in go.mod comments | Yes | **NO** | **GAP in JS** |
| go.mod parsing | Manual regex/string | tree-sitter WASM | Different mechanism |
| Toolchain entry filtering | `isGoToolchainEntry()` filters both `go@` and `toolchain@` | Only filters `go@` (missing `toolchain@`) | **GAP in JS** (TC-3818 secondary) |

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
| Workspace member discovery (batch) | **NO** | Yes | **GAP in Java** |

### Python (requirements.txt + pyproject.toml)

See `parity-report-python-2026-04-14.md` for full detail.

| Feature | Java | JS | Parity |
|---|---|---|---|
| pip (requirements.txt) | Yes | Yes | Parity (minor gaps) |
| pip (pyproject.toml / PEP 621) | Yes | Yes | Parity (minor gaps) |
| Poetry | **REJECTED** | Yes | **CRITICAL GAP** |
| uv | **NOT SUPPORTED** | Yes | **CRITICAL GAP** |

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
| pip (requirements.txt) | YES | YES | YES | **NO** |
| pyproject.toml | YES | YES | YES | YES |

### Wrapper Support

| Tool | Java | JS |
|---|---|---|
| mvnw (Maven) | YES | YES |
| gradlew (Gradle) | **NO** | YES |

### Workspace Support

| Ecosystem | Java Lock Walk-up | JS Lock Walk-up | Java `TRUSTIFY_WS_DIR` | JS `TRUSTIFY_WS_DIR` |
|---|---|---|---|---|
| npm | NO | YES | NO | YES |
| pnpm | NO | YES | NO | YES |
| yarn | NO | YES | NO | YES |
| Cargo | YES | YES | NO | YES |
| Maven | N/A (CLI) | N/A (CLI) | NO | NO |
| Gradle | N/A (CLI) | N/A (CLI) | NO | NO |
| Go | NO | NO | NO | NO |

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
| pyproject.toml dispatch | Single provider (`PythonPyprojectProvider`) | Provider chain: Poetry -> uv -> pip fallback |
| Ignore pattern detection | Centralized `IgnorePatternDetector` utility (both markers) | Per-ecosystem, inconsistent (some have both markers, most only have `exhortignore`) |
| Binary/tool selection | `Operations.getExecutable()`, `Operations.getCustomPathOrElse()` | `getCustomPath()` from `tools.js` |
| Manifest parsing | tree-sitter for Go; XML streaming for Maven; regex for others | tree-sitter for Go, pip; XML parser lib for Maven; regex for others |

---

## IDE Extension Support (not yet analyzed)

| Feature | IntelliJ | VS Code | Status |
|---|---|---|---|
| Diagnostics | TBD | TBD | Not yet analyzed |
| Quick fixes | TBD | TBD | Not yet analyzed |
| Vulnerability report | TBD | TBD | Not yet analyzed |

---

## Recommendations (Priority Order)

### 1. Fix `trustify-da-ignore` in JS client (ALL ecosystems)
Create a centralized ignore marker utility in JS (similar to Java's `IgnorePatternDetector`) that checks both `exhortignore` and `trustify-da-ignore`. Update: Maven (`java_maven.js`), Gradle (`java_gradle.js`), Base_javascript (`manifest.js`), Go (`golang_gomodules.js`), pip (`requirements_parser.js`).

### 2. Add `gradlew` wrapper support to Java client
The Java `GradleProvider` should use the same wrapper traversal pattern already implemented in `JavaMavenProvider.traverseForMvnw()`. Extract shared wrapper logic.

### 3. Add `go mod edit -json` to Java Go provider
The Java client should use `go mod edit -json` to correctly identify direct vs indirect dependencies, matching the JS client behavior. This will improve component analysis accuracy.

### 4. Add lock file walk-up for JS ecosystems in Java client
`JavaScriptProvider.validateLockFile()` needs directory walk-up. Highest impact for IntelliJ plugin users with npm/pnpm/yarn workspaces.

### 5. Add `TRUSTIFY_DA_MVN_USER_SETTINGS` and `TRUSTIFY_DA_MVN_LOCAL_REPO` to JS client
Copy the pattern from the Java client's `Operations.getMavenConfig()`.

### 6. Add `TRUSTIFY_DA_MVN_ARGS` to Java client
Copy the pattern from the JS client's `getCustom('TRUSTIFY_DA_MVN_ARGS')`.

### 7. Add Go root module version from git to JS client
Port the Java client's `determineMainModuleVersion()` VCS logic, or use `go list -m -json` as a simpler alternative.

### 8. Unify pnpm lock file update command
Decide on `--lockfile-only` vs `--frozen-lockfile` semantics. These have different behaviors when the lock file is stale.
