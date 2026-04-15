# requirements.txt Parsing Architecture (tree-sitter-based)

## Provider Entry Point

`src/providers/python_pip.js` handles `requirements.txt` manifests:
- `isSupported()` matches the exact filename `requirements.txt`
- Exports `provideStack()` and `provideComponent()` which both return `{ ecosystem: 'pip', content: <CycloneDX JSON>, contentType: 'application/vnd.cyclonedx+json' }`
- Provider is registered as a plain object (not a class instance) in `src/provider.js` via `availableProviders`
- `validateLockFile()` always returns `true` — pip has no lock file concept

## Tree-Sitter Integration

### WASM Grammar
- Grammar: `Strum355/tree-sitter-requirements` (commit `d0261ee76b84253997fe70d7d397e78c006c3801`)
- WASM file copied from `node_modules/tree-sitter-requirements/tree-sitter-requirements.wasm` to `src/providers/tree-sitter-requirements.wasm` during `pretest` and `postcompile` npm scripts
- Loaded via `web-tree-sitter` (JS WASM binding for tree-sitter)

### Parser Initialization (`src/providers/requirements_parser.js`)
- `init()`: calls `Parser.init()` (one-time WASM runtime init), reads the `.wasm` file as bytes, loads via `Language.load()`
- **Every exported function re-runs `init()`** — no caching of the Language object between calls. This is a design choice for simplicity (the WASM init has internal dedup).
- Exports four functions, each returning a Promise:
  - `getParser()` — returns a `Parser` configured with the requirements language
  - `getRequirementQuery()` — extracts all requirements: `(requirement (package) @name) @req`
  - `getIgnoreQuery()` — extracts requirements followed by `#exhortignore` comment: `((requirement (package) @name) @req . (comment) @comment (#match? @comment "^#[\\t ]*exhortignore"))`
  - `getPinnedVersionQuery()` — extracts pinned versions (using `==`): `(version_spec (version_cmp) @cmp (version) @version (#eq? @cmp "=="))`

### Grammar Node Types (key ones)
- `file` (root) — children: `requirement`, `comment`, `global_opt`, `path`, `url`
- `requirement` — children: `package`, `extras`, `version_spec`, `marker_spec`, `url_spec`, `requirement_opt`
- `version_spec` — children: `version_cmp` (e.g., `==`, `>=`), `version`
- `marker_spec` — environment markers (e.g., `; python_version < '3.11'`)
- `extras` — optional extras (e.g., `[security]`)
- `global_opt` — global options like `--extra-index-url`, `-e` (editable)
- `requirement_opt` — per-requirement options like `--config-settings`
- `comment` — standalone or inline comments

## What Tree-Sitter Extracts

### For Dependency Resolution (`#parseRequirements` in `python_controller.js`)
- Uses `getRequirementQuery()` to find all `(requirement (package) @name) @req` patterns
- For each match, extracts:
  - `name`: the `@name` capture from the `package` node (e.g., `flask`, `requests`)
  - `version`: runs `getPinnedVersionQuery()` on the requirement node to find `==X.Y.Z` pins; returns `null` if no pinned version
- **Only extracts `==` pinned versions** — other version specifiers (`>=`, `~=`, `!=`) are ignored at parsing level. Version resolution happens via pip itself.

### For Ignore Detection (`getIgnoredDependencies` in `python_pip.js`)
- Uses `getIgnoreQuery()` to find requirements immediately followed by `#exhortignore` or `# exhortignore` comments
- For each ignored dep, also runs `getPinnedVersionQuery()` to get the pinned version if present
- Returns `PackageURL[]` with `type=pypi`, name, and optional version

## Dependency Resolution Flow

### Stack Analysis (`createSbomStackAnalysis`)
1. Detect python/pip binaries (`python3`/`pip3` with fallback to `python`/`pip`)
2. Create `Python_controller` — optionally creates a virtual environment
3. `getDependencies(true)` — `includeTransitive=true`:
   - If virtual env: installs requirements, then queries installed packages
   - Runs `pip freeze --all` to get installed package list
   - Runs `pip show <all-deps>` to get metadata including `Requires:` field
   - OR uses `pipdeptree --json` if `TRUSTIFY_DA_PIP_USE_DEP_TREE=true`
   - Parses requirements.txt via tree-sitter to get declared deps
   - Recursively resolves transitive deps from pip show/pipdeptree data
   - Version mismatch check: if `MATCH_MANIFEST_VERSIONS=true` (default), verifies installed version matches manifest pinned version
4. Builds SBOM with root component `default-pip-root@0.0.0` + full dependency tree
5. Filters ignored deps (exhortignore)

### Component Analysis (`getSbomForComponentAnalysis`)
1. Same binary/environment setup
2. `getDependencies(false)` — `includeTransitive=false`:
   - Same pip freeze/show data gathering
   - But skips recursive transitive resolution — only direct deps
3. Builds SBOM with root + flat dependency list (no tree)
4. Filters ignored deps

### Key Difference
- Stack: full recursive dependency tree with `addAllDependencies()` building nested SBOM relationships
- Component: flat list, each dep added directly under root with `sbom.addDependency(rootPurl, toPurl(dep.name, dep.version))`

## SBOM Construction

- Uses `Sbom` wrapper around `CycloneDxSbom` (spec version 1.4)
- Root component: `pkg:pypi/default-pip-root@0.0.0` (type=application)
- Dependencies: `pkg:pypi/<name>@<version>` (type=library)
- No namespace for pypi PURLs (PackageURL namespace is `undefined`)
- Dependencies sorted alphabetically (case-insensitive)
- Circular dependency protection via `path` array tracking in `bringAllDependencies()`

## Edge Cases Handled

1. **`#exhortignore` comments** — tree-sitter query detects `#exhortignore` or `# exhortignore` (space-tolerant regex). Both `MATCH_MANIFEST_VERSIONS=true` (version-exact filtering) and `false` (name-only filtering) paths supported.
2. **Environment markers** (e.g., `; python_version < '3.11'`) — parsed by grammar as `marker_spec` but NOT used by the JS client. The grammar recognizes them, they just don't affect which packages are extracted.
3. **Per-requirement options** (e.g., `--config-settings=...`) — parsed by grammar as `requirement_opt`, not used by client.
4. **Global options** (e.g., `--extra-index-url`) — parsed by grammar as `global_opt`, not used by pip provider (but used by `python_uv.js` for `-e` editable installs).
5. **Line continuations** (backslash `\`) — handled by the grammar's `linebreak` node type.
6. **Name normalization** — `python_controller.js` caches deps with both hyphen and underscore variants (e.g., `importlib-metadata` and `importlib_metadata`), but only does single replacement (not exhaustive PEP 503 normalization).
7. **Virtual environment** — when `TRUSTIFY_DA_PYTHON_VIRTUAL_ENV=true`, creates venv at `/tmp/trustify_da_env_js`, installs requirements, queries, then uninstalls.
8. **Best-effort installation** — `TRUSTIFY_DA_PYTHON_INSTALL_BEST_EFFORTS=true` installs packages one-by-one without version pins, requires `MATCH_MANIFEST_VERSIONS=false`.
9. **pipdeptree alternative** — `TRUSTIFY_DA_PIP_USE_DEP_TREE=true` uses pipdeptree JSON output instead of pip freeze+show.

## NOT Handled
- `-r` includes (recursive file includes) — the grammar may parse paths but the provider does not follow them
- `-e` editable installs — only handled by `python_uv.js` (for `uv export` output), not by `python_pip.js`
- `--hash` options — grammar parses them but they are not used
- URL-based requirements (e.g., `package @ https://...`) — grammar has `url_spec` but provider ignores
- Extras (e.g., `requests[security]`) — grammar has `extras` node but provider extracts only package name

## Shared Parser Usage
- `python_pip.js` — uses `getParser()`, `getIgnoreQuery()`, `getPinnedVersionQuery()`
- `python_controller.js` — uses `getParser()`, `getRequirementQuery()`, `getPinnedVersionQuery()`
- `python_uv.js` — uses `getParser()`, `getPinnedVersionQuery()` (parses `uv export --format requirements.txt` output)
