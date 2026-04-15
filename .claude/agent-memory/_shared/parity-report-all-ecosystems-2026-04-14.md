# All-Ecosystems Parity Report -- 2026-04-14

## Executive Summary

This report covers dependency tree generation parity across ALL non-Python ecosystems for the Java and JavaScript API clients. Python analysis is in a separate report (`parity-report-python-2026-04-14.md`).

**Critical parity gaps found:**
1. **Ignore markers**: JS client only recognizes `exhortignore` for Maven, Gradle, Go, and npm/pnpm/yarn (package.json). Java client recognizes BOTH `exhortignore` AND `trustify-da-ignore` for all ecosystems. Only Cargo and pyproject.toml are at parity.
2. **Gradle wrapper**: Java client has NO `gradlew` wrapper support; JS client has full wrapper traversal. Java Maven has wrapper support; this is an inconsistency within the Java client itself.
3. **Go direct dependency identification**: Java uses `go mod graph` edge counting (all root edges = direct). JS uses `go mod edit -json` with `Indirect` flag to correctly classify direct vs transitive deps.
4. **Go main module version**: Java uses git tag/pseudo-version detection via VCS. JS hardcodes `v0.0.0` for the root module version.
5. **Maven settings.xml**: Java supports `TRUSTIFY_DA_MVN_USER_SETTINGS` and `TRUSTIFY_DA_MVN_LOCAL_REPO`. JS has no equivalent.
6. **Maven extra args**: JS supports `TRUSTIFY_DA_MVN_ARGS` for custom CLI args. Java has no equivalent.
7. **npm/pnpm/yarn workspace walk-up**: JS has full lock file walk-up and workspace detection. Java checks only the manifest directory.

---

## 1. Maven (pom.xml)

### A. Manifest Detection

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Filename match | `"pom.xml"` in `Ecosystem.resolveProvider()` | `isSupported('pom.xml')` in `Java_maven` | MATCH |
| Provider class | `JavaMavenProvider` | `Java_maven extends Base_Java` | MATCH |
| Lock file validation | No lock file concept (`validateLockFile` not enforced) | `validateLockFile()` returns `true` (no lock file) | MATCH |

### B. CLI Commands & Binary Delegation

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Binary | `mvn` via `Operations.getExecutable("mvn")` | `mvn` via `getCustomPath('mvn', opts)` | MATCH |
| Binary env var | `TRUSTIFY_DA_MVN_PATH` | `TRUSTIFY_DA_MVN_PATH` | MATCH |
| Wrapper preference | `Operations.getWrapperPreference("mvn")` -> `traverseForMvnw()` up to git root | `getWrapperPreference('mvn', opts)` -> `traverseForWrapper()` up to git root | MATCH |
| Wrapper name | `mvnw` (or `mvnw.cmd` on Windows) | `mvnw` (or `mvnw.cmd` on Windows) | MATCH |
| Stack: clean | `mvn clean -f pom.xml --batch-mode -q` | `mvn -q clean [MVN_ARGS]` | Minor diff: Java uses `-f`, `--batch-mode`; JS uses cwd |
| Stack: tree | `mvn org.apache.maven.plugins:maven-dependency-plugin:3.6.0:tree -Dscope=compile -Dverbose -DoutputType=text -DoutputFile=<tmp>` | Same plugin, same args | MATCH |
| Component: effective POM | `mvn clean help:effective-pom -Doutput=<tmp>` | `mvn -q help:effective-pom -Doutput=<tmp>` | MATCH |
| Settings.xml support | `TRUSTIFY_DA_MVN_USER_SETTINGS` -> `-s <path>` | **NOT SUPPORTED** | **GAP in JS** |
| Local repo support | `TRUSTIFY_DA_MVN_LOCAL_REPO` -> `-Dmaven.repo.local=<path>` | **NOT SUPPORTED** | **GAP in JS** |
| Custom extra args | **NOT SUPPORTED** | `TRUSTIFY_DA_MVN_ARGS` (JSON array) | **GAP in Java** |
| JAVA_HOME support | Yes (`JAVA_HOME` env var passed to mvn process) | Not explicitly (inherits from parent process) | Minor diff |

### C. Dependency Tree Construction

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Stack analysis | Parse `mvn dependency:tree` text output -> `parseDependencyTree()` (recursive, indentation-based) | Same: `parseDependencyTree()` in `Base_Java` (inherited) | MATCH |
| Component analysis | Parse `mvn help:effective-pom` XML -> extract all non-test deps as flat list | Same: parse effective POM -> flat dep list | MATCH |
| Multi-module support | Delegated to Maven CLI (aggregator POM detection via `packaging=pom` in XML) | Same: aggregator POM detection via `packaging === 'pom'` in parsed JSON | MATCH |
| Root component | From `<project>` element in effective POM: `groupId:artifactId:version` | From `#getRootFromPom()`: same fields | MATCH |
| PURL type | `maven` | `maven` | MATCH |
| PURL structure | `pkg:maven/groupId/artifactId@version` | Same | MATCH |
| Test dep filtering | Java: `DependencyAggregator.isTestDependency()` filters `scope=test` | JS: `dep['scope'] !== 'test'` filter | MATCH |

### D. Ignored Dependencies

| Aspect | Java | JS | Parity |
|---|---|---|---|
| POM XML parsing | XMLStreamReader (streaming) | fast-xml-parser (`XMLParser`) | Different mechanism, same result |
| Ignore markers | `exhortignore` AND `trustify-da-ignore` (via `isIgnoreComment()` -> `IgnorePatternDetector`) | **Only `exhortignore`** (hardcoded: `dep['#comment'].includes('exhortignore')`) | **GAP in JS** |
| dependencyManagement skipping | Yes (tracks `insideDependencyManagement` state) | Not explicitly -- parses `project.dependencies.dependency` path only | Likely MATCH (fast-xml-parser jpath filter) |
| plugins skipping | Yes (tracks `insidePlugins` state) | Not explicitly | Possible minor diff |
| Exclusion handling | Ignored deps passed to `-Dexcludes=groupId:artifactId` | Same: ignored deps collected and passed as `-Dexcludes=` | MATCH |

### E. SBOM Generation

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Format | CycloneDX JSON | CycloneDX JSON | MATCH |
| Content type | `application/vnd.cyclonedx+json` | `application/vnd.cyclonedx+json` | MATCH |
| Root: stack | From dep tree first line (parsed from `groupId:artifactId:packaging:version`) | Same | MATCH |
| Root: component | From effective POM XML project element | From effective POM parsed JSON | MATCH |
| Sbom matching | Coordinate-based (`PURL`, `sensitive`) for stack | Default SBOM factory | Minor diff in factory options |

### F. License Reading

| Aspect | Java | JS | Parity |
|---|---|---|---|
| License source | `readLicenseFromPom()`: reads `<licenses><license><name>` from raw pom.xml | `readLicenseFromManifest()`: reads `project.licenses.license.name` from parsed XML | MATCH |
| License fallback | `LicenseUtils.readLicenseFile()` | `getLicense()` | MATCH |

### G. Test Fixtures

| Feature | Java | JS | Parity |
|---|---|---|---|
| No ignore | `deps_with_no_ignore` | `pom_deps_with_no_ignore` | MATCH |
| With ignore | `deps_with_ignore_on_artifact/dependency/group/version/wrong` | `pom_deps_with_ignore_on_artifact/dependency/group/version/wrong` | MATCH |
| Multi-module | None explicitly | `pom_with_multiple_modules`, `pom_with_one_module` | **JS has more** |
| Wrapper | None explicitly | `pom_with_mvn_wrapper` | **JS has more** |
| License | `license/` | None explicitly | **Java has more** |
| Long deps | None | `poms_deps_with_2_ignore_long`, `poms_deps_with_ignore_long`, `poms_deps_with_no_ignore_long` | **JS has more** |
| Common paths | `pom_deps_with_no_ignore_common_paths` | `pom_deps_with_no_ignore_common_paths` | MATCH |
| Version from property | None | `pom_deps_with_ignore_version_from_property` | **JS has more** |

---

## 2. Gradle (build.gradle / build.gradle.kts)

### A. Manifest Detection

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Filename match | `"build.gradle"` and `"build.gradle.kts"` in `Ecosystem.resolveProvider()` -> same `GradleProvider` | `java_gradle_groovy.js` matches `build.gradle`, `java_gradle_kotlin.js` matches `build.gradle.kts` | MATCH (both produce same behavior) |
| Groovy vs Kotlin | Single `GradleProvider` handles both; no subclass distinction | `Java_gradle_groovy extends Java_gradle`, `Java_gradle_kotlin extends Java_gradle` -- subclasses override `_extractDepToBeIgnored`, `_getManifestName`, `_parseAliasForLibsNotation` | Different architecture, functionally equivalent |
| Lock file validation | None (no lock file) | `validateLockFile()` returns `true` | MATCH |

### B. CLI Commands & Binary Delegation

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Binary | `gradle` via `Operations.getExecutable("gradle", "--version")` -- no wrapper! | `gradle` via `getCustomPath('gradle', opts)` + **wrapper support via `selectToolBinary()` -> `traverseForWrapper()`** | **GAP in Java** -- no `gradlew` support |
| Binary env var | `TRUSTIFY_DA_GRADLE_PATH` | `TRUSTIFY_DA_GRADLE_PATH` | MATCH |
| Wrapper support | **NO** -- `GradleProvider` directly uses `Operations.getExecutable("gradle")` with no wrapper traversal | **YES** -- `Base_Java.selectToolBinary()` traverses for `gradlew` (or `gradlew.bat` on Windows) up to git root | **GAP in Java** |
| Command: dependencies | `gradle dependencies` | `gradle dependencies` (via `_invokeCommand(gradle, ['dependencies'])`) | MATCH |
| Command: properties | `gradle properties` (for root project name, group, version) | `gradle properties` (via `_invokeCommand`) | MATCH |
| Timeout | 120 seconds (`TIMEOUT = 120`) | Inherits from `invokeCommand()` default | Minor diff |

### C. Dependency Tree Construction

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Stack analysis | Parse `gradle dependencies` output -> extract `runtimeClasspath` and `compileClasspath` configurations -> `parseDependencyTree()` (recursive, indentation-based) | Same: `#extractConfigurations()` -> `#processDependencyTree()` | MATCH |
| Component analysis | Same parse, but filter to `depth==1` only (direct deps) using `ProcessedLine` | Same: `#buildDirectDependenciesSbom()` -> `#processDirectDependencies()` | MATCH |
| Configuration handling | Runtime = REQUIRED scope, Compile = OPTIONAL scope | Runtime = 'required', Compile = 'optional' | MATCH |
| Dedup handling | Not explicit (parseDependencyTree handles) | `processedDeps` Set prevents duplicates | JS more explicit but same result |
| Root component | From `gradle properties`: `group:rootProjectName:jar:version` | Same: `${properties.group}:${rootProjectName}:jar:${properties.version}` | MATCH |
| Version catalog (libs.versions.toml) | `getLibsVersionsTomlPath()` -> resolves `gradle/libs.versions.toml` relative to manifest | `#getDepFromLibsNotation()` -> resolves `gradle/libs.versions.toml` | MATCH |

### D. Ignored Dependencies

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Ignore markers | `exhortignore` AND `trustify-da-ignore` (via `IgnorePatternDetector.containsIgnorePattern()`) | **Only `exhortignore`** (regex: `TRUSTIFY_DA_IGNORE_REGEX_LINE = /.*\s?exhortignore\s*$/g`) | **GAP in JS** |
| Comment styles | `//` and `/* */` comments stripped before extracting dep | Same: `//` and `/* */` handling | MATCH |
| libs notation | Resolves aliases via `getDepFromNotation()` -> reads `libs.versions.toml` | Same: `#getDepFromLibsNotation()` -> parses TOML | MATCH |
| Groovy string dep | Extracted via regex (Java handles both Groovy/Kotlin in one class) | `GROOVY_DEP_REGEX` in `java_gradle_groovy.js` | MATCH |
| Kotlin dep | Same class | `KOTLIN_DEP_REGEX` in `java_gradle_kotlin.js` | MATCH |

### E. SBOM Generation

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Root PURL | `pkg:maven/group/name@version` (type=jar in coordinate, but PURL type is maven) | Same | MATCH |
| Scope qualifiers | `REQUIRED` and `OPTIONAL` on components | `required` and `optional` | MATCH |
| Ignored dep filtering | `filterIgnoredDeps(ignored)` (coordinate-based) | `filterIgnoredDepsIncludingVersion(ignoredDeps)` (version-aware) | **Possibly different**: JS uses version-aware filtering |

### G. Test Fixtures

| Feature | Java (Groovy + Kotlin) | JS | Parity |
|---|---|---|---|
| No ignore | `deps_with_no_ignore_common_paths` | `deps_with_no_ignore_common_paths` | MATCH |
| Full spec ignore | `deps_with_ignore_full_specification` | `deps_with_ignore_full_specification` | MATCH |
| Named params | `deps_with_ignore_named_params` | `deps_with_ignore_named_params` | MATCH |
| Notations | `deps_with_ignore_notations` | `deps_with_ignore_notations` | MATCH |
| Duplicate versions | `deps_with_duplicate_different_versions`, `deps_with_duplicate_no_version` (Groovy+Kotlin only) | None | **Java has more** |
| Empty project | `empty` (Groovy+Kotlin) | `deps_with_empty_project_group` | Different name, likely similar |

---

## 3. npm (package.json + package-lock.json)

### A. Manifest Detection

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Filename match | `"package.json"` in `Ecosystem.resolveProvider()` -> `JavaScriptProviderFactory.create()` | `isSupported('package.json')` in `Base_javascript` | MATCH |
| Lock file match | `package-lock.json` checked in manifest dir only | `package-lock.json` via `_findLockFileDir()` with walk-up | **GAP in Java** |
| Provider dispatch | `JavaScriptProviderFactory` iterates lock files in order: `package-lock.json` -> `pnpm-lock.yaml` -> `yarn.lock` | Provider array: `javascript_npm`, `javascript_pnpm`, `javascript_yarn` each with `validateLockFile()` | MATCH (same priority) |

### B. CLI Commands

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Binary | `npm` via `Operations.getExecutable("npm", "-v")` | `npm` via `getCustomPath('npm', opts)` | MATCH |
| Binary env var | `TRUSTIFY_DA_NPM_PATH` (via `ENV_NODE_HOME` pattern) | `TRUSTIFY_DA_NPM_PATH` | MATCH |
| Update lock | `npm i --package-lock-only` | `npm install --package-lock-only` | MATCH |
| List (stack) | `npm ls --all --omit=dev --package-lock-only --json` | Same args | MATCH |
| List (component) | `npm ls --depth=0 --omit=dev --package-lock-only --json` | Same args | MATCH |
| PATH augmentation | `pathEnv()` -> `TRUSTIFY_DA_NPM_PATH` appended to PATH | fnm/nvm/local `node_modules/.bin` auto-detection + PATH augmentation | **JS has richer** version manager integration |

### C. Dependency Tree Construction

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Tree source | `npm ls --json` output parsed as JSON tree | Same | MATCH |
| Tree traversal | `addDependenciesOf()` recursive on `dependencies` key | `#addDependenciesOf()` recursive on `dependencies` key | MATCH |
| Direct dep identification | From `dependencies`, `peerDependencies`, `optionalDependencies` in manifest | Same (via `Manifest.loadDependencies()`) | MATCH |
| Peer/optional deps | `ensurePeerAndOptionalDeps()` adds them if present in tree but missing from sbom | `#ensurePeerAndOptionalDeps()` same logic | MATCH |
| Root component | `name` and `version` from `package.json` | Same | MATCH |
| PURL type | `npm` | `npm` | MATCH |
| Scoped packages | Handled: `@scope/name` -> namespace=`@scope`, name=`name` | Same | MATCH |

### D. Ignored Dependencies

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Mechanism | `exhortignore` JSON array in `package.json` (checks both `trustify-da-ignore` and `exhortignore` fields) | `exhortignore` JSON array in `package.json` (**only** `exhortignore` field) | **GAP in JS** |
| Java field lookup | `content.get("trustify-da-ignore")` first, falls back to `content.get("exhortignore")` | Only `content.exhortignore` | **GAP in JS** |

### E. Transitive Dependency Count Bugs (TC-3818)

**Reported by Matej Nesuta (2026-03-23)**: JS client reports 196 total (187 transitive) vs Java's 193 total (184 transitive) for `package_json_deps_with_exhortignore_object` manifest. Same discrepancy affects pnpm and yarn.

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Version null check at root | `addDependenciesFromKey()` (line 180-183): skips entry AND entire subtree if `versionNode == null` | `_addDependenciesToSbom()`: no version check, processes versionless entries and recurses into children | **BUG in Java** — under-counts for `file:` deps, workspace packages, linked packages |
| `filterIgnoredDeps` behavior | `CycloneDXSbom.filterIgnoredDeps()`: recursive removal of ignored dep + all transitive-only children (insensitive mode via `TRUSTIFY_DA_IGNORE_METHOD`) | `CycloneDxSbom.filterIgnoredDeps()` (lines 214-235): non-recursive, removes only the directly ignored component, orphaned transitives remain in SBOM | **BUG in JS** — over-counts when `exhortignore` is used (orphaned deps not cleaned up) |
| `TRUSTIFY_DA_IGNORE_METHOD` env var | Supported (`sensitive`/`insensitive`, default `insensitive`) | **NOT SUPPORTED** | **GAP in JS** |

**Note**: Both bugs contribute to the discrepancy but in opposite directions. The Java client under-counts (null version skip) while the JS client over-counts (non-recursive ignore removal). The net effect depends on the specific manifest.

### F. SBOM Generation

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Stack | Full tree with transitives from `npm ls --all --json` | Same | MATCH |
| Component | Direct deps only from `npm ls --depth=0 --json`, filtered to manifest deps | Same | MATCH |

### F. Workspace Support

Covered in detail in the parity matrix. **Summary: JS has full walk-up + `TRUSTIFY_DA_TRUSTIFY_WS_DIR` + workspace root detection; Java has none.**

---

## 4. pnpm (package.json + pnpm-lock.yaml)

### Differences from npm (only listing what differs)

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Lock file | `pnpm-lock.yaml` checked in manifest dir only | `pnpm-lock.yaml` via walk-up + `pnpm-workspace.yaml` boundary | **GAP in Java** |
| List (stack) | `pnpm ls --dir <dir> --depth=Infinity --prod --json` | `pnpm ls --depth=Infinity --prod --json` | MATCH (minor: Java passes `--dir`) |
| Array output | `buildDependencyTree()` overrides to return `depTree.get(0)` | `_buildDependencyTree()` overrides to return `tree[0]` | MATCH |
| Update lock | `pnpm i --lockfile-only` | `pnpm install --frozen-lockfile` | **DIFFERENT**: Java uses `--lockfile-only`, JS uses `--frozen-lockfile` |
| Ignore markers | `exhortignore` AND `trustify-da-ignore` (via Manifest class) | **Only `exhortignore`** | **GAP in JS** |
| Workspace | No walk-up, no `pnpm-workspace.yaml` | Full walk-up + `pnpm-workspace.yaml` detection | **GAP in Java** |

---

## 5. Yarn Classic (package.json + yarn.lock, Yarn v1)

### Differences from npm

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Version detection | `yarn -v` -> regex `^([0-9]+)\.` -> major=1 means classic | Same: `yarn --version` -> regex parse | MATCH |
| Lock file | `yarn.lock` checked in manifest dir only | `yarn.lock` via walk-up + workspace root detection | **GAP in Java** |
| List (stack) | `yarn list --depth Infinity --prod --frozen-lockfile --json` | `yarn list --depth=Infinity --prod --frozen-lockfile --json` | Minor: `--depth Infinity` (Java, space) vs `--depth=Infinity` (JS, equals) |
| List (component) | `yarn list --depth 0 --prod --frozen-lockfile --json` | `yarn list --depth=0 --prod --frozen-lockfile --json` | Same minor diff |
| Output parsing | `parseDepTreeOutput` strips `yarn list` output header to get JSON | Same | MATCH |
| Tree structure | `getRootDependencies()` reads `data.trees` array from yarn JSON | Same | MATCH |
| Dep tree traversal | `addDependenciesToSbom()` -> `addChildrenToSbom()` recursive on `children` | Same | MATCH |
| Ignore markers | `exhortignore` AND `trustify-da-ignore` | **Only `exhortignore`** | **GAP in JS** |

---

## 6. Yarn Berry (package.json + yarn.lock, Yarn v2+)

### Differences from npm

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Version detection | Same as Classic (major >= 2 means Berry) | Same | MATCH |
| List (stack) | `yarn info --recursive --json` | Same | MATCH |
| List (component) | `yarn info --all --json` | Same | MATCH |
| Output parsing | `parseDepTreeOutput()` -> NDJSON -> JSON array wrapping | Same | MATCH |
| Root detection | `isRoot()` -> `name.endsWith("@workspace:.")` | `#isRoot()` -> same check | MATCH |
| Locator patterns | `LOCATOR_PATTERN`, `VIRTUAL_LOCATOR_PATTERN` for `name@npm:version` and `name@virtual:hash#npm:version` | Same patterns | MATCH |
| SBOM build | Complex graph traversal: builds node map, resolves virtual locators, tracks dependencies | Same logic | MATCH |
| Ignore markers | `exhortignore` AND `trustify-da-ignore` | **Only `exhortignore`** | **GAP in JS** |
| Workspace support | `@workspace:.` root detection only | Same + lock file walk-up + workspace member discovery | **GAP in Java** |

---

## 7. Go Modules (go.mod + go.sum)

### A. Manifest Detection

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Filename match | `"go.mod"` in `Ecosystem.resolveProvider()` | `isSupported('go.mod')` | MATCH |
| Lock file | No lock file validation (go.sum not checked) | `validateLockFile()` returns `true` (no validation) | MATCH |

### B. CLI Commands

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Binary | `go` via `Operations.getExecutable("go", "version")` | `go` via `getCustomPath('go', opts)` | MATCH |
| Binary env var | `TRUSTIFY_DA_GO_PATH` | `TRUSTIFY_DA_GO_PATH` | MATCH |
| Command: graph | `go mod graph` | `go mod graph` | MATCH |
| Command: edit | **NOT USED** | `go mod edit -json` | **GAP in Java** |
| MVS logic | `TRUSTIFY_DA_GO_MVS_LOGIC_ENABLED` (default `true`) | Same env var (default `"true"`) | MATCH |

### C. Dependency Tree Construction -- **SIGNIFICANT DIFFERENCES**

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Root module path | Extracted from first line of `go mod graph` output: `getParentVertex(linesList.get(0))` | From `go mod edit -json` -> `Module.Path` | **DIFFERENT** approach |
| Root module version | **Git VCS detection**: `determineMainModuleVersion()` uses git tags, pseudo-version generation. Defaults to `v0.0.0`. | **Hardcoded** `v0.0.0` (`defaultMainModuleVersion`) | **GAP in JS** -- Java has richer version detection |
| Direct dep identification (stack) | All edges from root in `go mod graph` treated as direct deps of root | Uses `go mod edit -json` `Require` field with `Indirect: false` to identify true direct deps; skips `go mod graph` root edges where the child's path is not in `directDepPaths` | **DIFFERENT** -- JS correctly excludes transitive deps promoted by MVS |
| Direct dep identification (component) | `collectAllDirectDependencies()`: edges from root in graph | Same as stack but filtered: `directDepPaths.has(getPackageName(child))` | **Same gap as above** |
| Go manifest parsing | `collectAllDepsFromManifest()`: manual regex/string parsing of `require` blocks and inline `require` statements from `go.mod` text | `collectAllDepsFromManifest()`: **tree-sitter** (`tree-sitter-go-mod.wasm`) query parsing | **DIFFERENT** mechanism |
| Graph building | Build `Map<String, List<String>>` edges from `go mod graph` lines — **BUG TC-3818**: `getFinalPackagesVersionsForModule()` uses `HashMap.put()` which overwrites children when multiple original parent versions resolve to the same MVS-selected version, silently dropping reachable transitive deps | Process `go mod graph` rows directly as flat array, all edges preserved | **BUG in Java** (TC-3818) — Java drops 6 transitive deps in test fixture |
| Toolchain filtering | `isGoToolchainEntry()` filters both `go@` AND `toolchain@` entries | Only filters `go@` children (`!line.includes(' go@')`) — misses `toolchain@` | **GAP in JS** (TC-3818) |

### D. Ignored Dependencies

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Ignore markers | `exhortignore` AND `trustify-da-ignore` (via `IgnoredLine()` with complex pattern matching) | **Only `exhortignore`** (via tree-sitter query + `hasExhortIgnore()` regex) | **GAP in JS** |
| Ignore validation | Complex rules: skips `module`, `go`, `require (`, `exclude`, `replace`, `retract`, `use`, `=>` lines | Tree-sitter structurally ensures only `require_spec` nodes are checked | MATCH (different approach, same intent) |
| Version-aware ignore | Collects versioned PURLs, then also filters by name for MVS-promoted versions | Same: `enforceRemovingIgnoredDepsInCaseOfAutomaticVersionUpdate()` | MATCH |

### E. MATCH_MANIFEST_VERSIONS

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Default | `false` | `"false"` | MATCH |
| Behavior | Compares manifest versions against `go mod graph` resolved versions | Same: `performManifestVersionsCheck()` | MATCH |

### F. Test Fixtures

| Feature | Java | JS | Parity |
|---|---|---|---|
| No ignore | `go_mod_light_no_ignore`, `go_mod_no_ignore` | `go_mod_light_no_ignore`, `go_mod_no_ignore` | MATCH |
| With ignore | `go_mod_with_ignore`, `go_mod_with_all_ignore`, `go_mod_with_one_ignored_prefix_go` | `go_mod_with_ignore`, `go_mod_with_all_ignore`, `go_mod_test_ignore` | Similar |
| MVS versions | None | `go_mod_mvs_versions` | **JS has more** |
| Empty | None | `go_mod_empty` | **JS has more** |
| No path | `go_mod_no_path` | None | **Java has more** |

---

## 8. Cargo/Rust (Cargo.toml + Cargo.lock)

### A. Manifest Detection

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Filename match | `"Cargo.toml"` in `Ecosystem.resolveProvider()` | `isSupported('Cargo.toml')` | MATCH |
| Lock file | `Cargo.lock` via walk-up (`findOutermostCargoTomlDirectory()` then check for Cargo.lock) | `Cargo.lock` via walk-up (`validateLockFile()` with `[workspace]` boundary) | MATCH (different strategy, same result) |

### B. CLI Commands

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Binary | `cargo` via `Operations.getExecutable("cargo", "--version")` | `cargo` via `getCustomPath('cargo', opts)` | MATCH |
| Binary env var | `TRUSTIFY_DA_CARGO_PATH` | `TRUSTIFY_DA_CARGO_PATH` | MATCH |
| Command | `cargo metadata --format-version 1` | Same | MATCH |
| Timeout | 120 seconds (with dedicated stream executor for deadlock avoidance) | Inherits from `invokeCommand()` default | Minor diff |
| Error handling | Bounded ExecutorService with proper stream reading to avoid buffer deadlocks | Simple `execSync`/`invokeCommand` | Java more robust |

### C. Dependency Tree Construction

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Metadata parsing | Jackson `ObjectMapper` -> `CargoMetadata` record | `JSON.parse()` -> raw object | MATCH |
| Crate type detection | `getProjectLayout()` -> `SINGLE_CRATE`, `WORKSPACE_VIRTUAL`, `WORKSPACE_WITH_ROOT_CRATE` | `detectCrateType()` -> same enum values | MATCH |
| Single crate handling | `handleSingleCrate()`: resolve root -> `processDirectDependencies()` -> recursive `processDependencyNode()` | `handleSingleCrate()`: same logic | MATCH |
| Virtual workspace | `handleVirtualWorkspace()`: iterate members, process each | Same | MATCH |
| Resolve graph | Uses `metadata.resolve().nodes()` to build node map | Same: `metadata.resolve.nodes` | MATCH |
| Package lookup | `buildPackageMap()`: Map of `id -> package` | `findPackageById()`: linear search | Minor perf diff |
| Path dependencies | `createCargoPurl()` with `repository_url=local` qualifier | `toPathDepPurl()` with `repository_url: 'local'` qualifier | MATCH |
| Runtime dep filtering | `shouldSkipDependency()`: filters `depKinds` for non-dev, non-build deps | `filterRuntimeDeps()`: same filtering on `dep_kinds` | MATCH |

### D. Ignored Dependencies

| Aspect | Java | JS | Parity |
|---|---|---|---|
| Ignore markers | `exhortignore` AND `trustify-da-ignore` (via `IgnorePatternDetector.containsIgnorePattern()`) | `exhortignore` AND `trustify-da-ignore` (via `IGNORE_MARKERS = ['exhortignore', 'trustify-da-ignore']`) | **MATCH** |
| Scanning scope | All `[dependencies]`, `[dev-dependencies]`, `[build-dependencies]` sections + member manifests | Same + workspace member scanning via `metadata.workspace_members` | MATCH |
| Line matching | `lineContainsDependency()`: checks `depName = ` or `[*.dependencies.depName]` patterns | `isInDependencySection()` + `isDepIgnored()`: similar patterns | MATCH |
| Workspace member ignore | `getMemberIgnoredDeps()`: reads each member's Cargo.toml | `getIgnoredDeps()`: iterates `workspace_members`, reads each member manifest | MATCH |

### E. SBOM Generation

| Aspect | Java | JS | Parity |
|---|---|---|---|
| PURL type | `cargo` | `cargo` | MATCH |
| Virtual workspace root | `name` from `[workspace.package]` or fallback, version=`0.0.0` | Same logic | MATCH |
| Workspace version inheritance | `version = { workspace = true }` -> reads `workspace.package.version` | `getWorkspaceVersion()` -> reads `parsed.workspace?.package?.version` | MATCH |
| `[workspace.dependencies]` | `processWorkspaceDependencies()`: reads from TOML table for component analysis | `getWorkspaceDepsFromManifest()`: reads from parsed TOML | MATCH |

### F. Test Fixtures

| Feature | Java | JS | Parity |
|---|---|---|---|
| Single crate no ignore | (in-memory tests) | `cargo_single_crate_no_ignore` | JS has fixture files |
| Single crate with ignore | (in-memory tests) | `cargo_single_crate_with_ignore`, `cargo_single_crate_with_exhortignore`, `cargo_single_crate_with_hyphen_ignore` | **JS has more** |
| Virtual workspace | (in-memory tests) | `cargo_virtual_workspace` + 6 variants | **JS has more** |
| Workspace with root | (in-memory tests) | `cargo_workspace_with_root` + 2 variants | **JS has more** |
| License | `license/` | `cargo_single_crate_with_license`, `cargo_virtual_workspace_with_license` | Both have |

**Cargo is the most aligned ecosystem between the two clients.** Both ignore markers are supported, workspace handling is functionally equivalent, and SBOM generation matches.

---

## Cross-Ecosystem Ignore Marker Summary

| Ecosystem | Java `trustify-da-ignore` | JS `trustify-da-ignore` | Parity |
|---|---|---|---|
| Maven (pom.xml) | YES | **NO** | **GAP in JS** |
| Gradle (build.gradle) | YES | **NO** | **GAP in JS** |
| npm (package.json) | YES (JSON field) | **NO** (only `exhortignore` field) | **GAP in JS** |
| pnpm (package.json) | YES | **NO** | **GAP in JS** |
| yarn (package.json) | YES | **NO** | **GAP in JS** |
| Go (go.mod) | YES | **NO** | **GAP in JS** |
| Cargo (Cargo.toml) | YES | YES | MATCH |
| pip (requirements.txt) | YES | **NO** (tree-sitter query) | **GAP in JS** |
| pyproject.toml | YES | YES | MATCH |

**Pattern**: The JS client has `trustify-da-ignore` support only in Cargo (`IGNORE_MARKERS`) and pyproject.toml (`IGNORE_MARKERS` in `base_pyproject.js`). All other ecosystems in JS only recognize the legacy `exhortignore` marker. The Java client uses the centralized `IgnorePatternDetector` utility that checks both markers for all ecosystems.

---

## Cross-Ecosystem Wrapper Support Summary

| Tool | Java Wrapper | JS Wrapper | Parity |
|---|---|---|---|
| Maven (`mvnw`) | YES (`traverseForMvnw()` in `JavaMavenProvider`) | YES (`traverseForWrapper()` in `Base_Java`) | MATCH |
| Gradle (`gradlew`) | **NO** (uses `Operations.getExecutable("gradle")` directly) | YES (`traverseForWrapper()` in `Base_Java`) | **GAP in Java** |

---

## Cross-Ecosystem Version Manager / PATH Support

| Feature | Java | JS | Parity |
|---|---|---|---|
| fnm integration | NO | YES (checks `FNM_DIR`) | **GAP in Java** |
| nvm integration | NO | YES (checks `NVM_DIR`) | **GAP in Java** |
| local `node_modules/.bin` | NO | YES (auto-detected) | **GAP in Java** |
| NODE_HOME / PATH | Basic `pathEnv()` support | Rich integration above | JS richer |

---

## Critical Parity Gaps Summary (ranked by impact)

### 0. TC-3818: Transitive dependency count discrepancies (Go, npm, pnpm, yarn)
**Impact**: Different transitive dependency counts between Java and JS clients across multiple ecosystems. Three independent root causes:
- **Go (Java bug)**: `GoModulesProvider.getFinalPackagesVersionsForModule()` line 357 — `HashMap.put()` overwrites children when MVS remaps multiple original versions to the same key. Fix: use `merge()` with `LinkedHashSet` to combine children.
- **npm/pnpm (Java bug)**: `JavaScriptProvider.addDependenciesFromKey()` lines 180-183 — skips entire dependency subtree when root-level dep has null version. Fix: still recurse into children even when version is missing.
- **npm/pnpm/yarn (JS bug)**: `CycloneDxSbom.filterIgnoredDeps()` lines 214-235 — non-recursive removal leaves orphaned transitive deps in SBOM. Fix: implement recursive removal matching Java's insensitive mode.
- **Go (JS bug, secondary)**: `golang_gomodules.js` line 249 — only filters `go@` children but not `toolchain@`. Fix: extend filter to match Java's `isGoToolchainEntry()`.

### 1. `trustify-da-ignore` marker in JS client (ALL ecosystems except Cargo + pyproject)
**Impact**: Users migrating from `exhortignore` to the new `trustify-da-ignore` marker will find their ignore annotations silently ignored by the JS client for Maven, Gradle, npm, pnpm, yarn, Go, and pip requirements.txt.

### 2. Gradle wrapper (`gradlew`) in Java client
**Impact**: Java client (used by IntelliJ plugin) cannot use `gradlew` wrappers, which is the standard way to invoke Gradle in most projects. Users with Gradle wrapper-only setups will fail.

### 3. Go direct dependency classification
**Impact**: Java client may include transitive dependencies promoted by MVS as "direct" deps in component analysis, producing larger SBOMs with false positives. JS client correctly uses `go mod edit -json` to classify.

### 4. JS ecosystem workspace walk-up in Java
**Impact**: IntelliJ plugin fails when analyzing `package.json` files in npm/pnpm/yarn workspace members because it cannot find the lock file at the workspace root.

### 5. Maven settings.xml in JS client
**Impact**: JS client (used by VS Code extension) cannot use custom Maven settings files for repository authentication or proxy configuration.

### 6. Maven extra args (`TRUSTIFY_DA_MVN_ARGS`) in Java client
**Impact**: Java client cannot pass custom Maven arguments (e.g., `-P profile`, `-Dproperty=value`).

### 7. Go main module version in JS client
**Impact**: JS client always uses `v0.0.0` for the root module, while Java client attempts to derive the correct version from git tags. This affects SBOM accuracy.

### 8. pnpm lock file update command
**Impact**: Different commands (`--lockfile-only` vs `--frozen-lockfile`) have different semantics: `--frozen-lockfile` fails if the lock file would need updating, while `--lockfile-only` updates it in place. This can cause different behavior when the lock file is out of date.
