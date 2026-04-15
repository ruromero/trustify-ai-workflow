# requirements.txt Parsing Architecture

## Entry Point

`Ecosystem.resolveProvider()` matches the filename `requirements.txt` and returns a `PythonPipProvider(manifestPath)`. This is the sole entry point. The provider hierarchy is:

```
Provider (abstract)
  -> PythonProvider (abstract) — shared Python infrastructure
       -> PythonPipProvider — requirements.txt handler
       -> PythonPyprojectProvider — pyproject.toml handler
```

## File Parsing Strategy

**No grammar or library.** The parser uses line-by-line string processing with indexOf-based extraction. There are two distinct parsing phases:

### Phase 1: Requirements File Reading (`PythonControllerBase.getDependenciesImpl`)

The requirements.txt file is read via `Files.readAllLines()` then filtered:
- Lines starting with `#` (comments) are dropped.
- Empty/blank lines are dropped.
- Each remaining line is trimmed.

Per line, the parser handles:
1. **PEP 508 environment markers** — detects `;` separator. If present, splits the line at `;` to get the requirement spec (left side). If the package is not found in the environment cache (marker didn't match the platform), the dependency is silently skipped.
2. **Version pinning** (`==`) — extracts dependency name and manifest version, then optionally validates against installed version (controlled by `MATCH_MANIFEST_VERSIONS` env var).
3. **Inline comments** — when extracting version from `==` lines, strips anything after `#` from the version string.

### Phase 2: Dependency Name Extraction (`PythonControllerBase.getDependencyName`)

Static method that extracts a package name by finding the first occurrence of `>`, `<`, or `=` in the requirement string, then taking the substring before it. If none of those characters exist, the entire string is the name. Also strips environment markers (`;`) first.

### Phase 3: Ignore Pattern Detection (`PythonPipProvider.getIgnoredDependencies`)

Re-reads the manifest content, splits by `System.lineSeparator()`, and filters for lines containing `#trustify-da-ignore` or `#exhortignore` (with optional space after `#`). For matched lines:
- `extractDepFull()` takes everything before the `#` comment
- `splitToNameVersion()` attempts to parse `name==version` using a regex; if it doesn't match, falls back to name-only with version `*`

The regex for version matching is: `[a-zA-Z0-9-_()]+={2}[0-9]{1,4}[.][0-9]{1,4}(([.][0-9]{1,4})|...)?`

## Dependency Resolution (Not Parsing)

The parser does NOT resolve dependencies from the requirements.txt file itself. Instead, it delegates to `pip` CLI tools for actual dependency resolution. Two strategies:

### Strategy A: pip freeze + pip show (default)

1. `pip freeze --all` to get all installed packages with versions
2. `pip show <all-dep-names>` to get dependency relationships (the `Requires:` field)
3. Parse pip show output by splitting on `---` delimiters between package blocks
4. Build a `HashMap<StringInsensitive, PythonDependency>` cache

### Strategy B: pipdeptree (opt-in via `TRUSTIFY_DA_PIP_USE_DEP_TREE=true`)

1. `pip install pipdeptree`
2. `pipdeptree --json` to get dependency tree as JSON
3. Parse using Jackson: extracts `package.package_name`, `package.installed_version`, and `dependencies[].package_name`

### Package name normalization

The cache stores each dependency under three key variations: original name, name with `-` replaced by `_`, and name with `_` replaced by `-`. All lookups are case-insensitive via `StringInsensitive` wrapper.

## SBOM Construction

### Stack Analysis (`provideStack`)

- Creates SBOM with `BelongingCondition.PURL` and `"sensitive"` ignore method
- Calls `getDependencies(manifest, includeTransitive=true)`
- Recursively walks the dependency tree via `addAllDependencies()`, adding each package as a `PackageURL` (type=pypi) dependency of its parent
- Root component: `default-pip-root@0.0.0`
- Then applies ignore filtering

### Component Analysis (`provideComponent`)

- Creates SBOM with default `BelongingCondition` (no sensitive mode)
- Calls `getDependencies(manifest, includeTransitive=false)`
- Adds only direct dependencies as flat list under root (no transitive tree)
- Root component: same `default-pip-root@0.0.0`
- Then applies ignore filtering

## Environment Modes

Three controller implementations:
- **RealEnv** — uses existing system Python/pip installation, does NOT auto-install packages
- **VirtualEnv** — creates a venv at `/tmp/trustify_da_env`, auto-installs packages, cleans up after
- **TestEnv** — extends RealEnv but auto-installs packages (upgrades pip first)

Selection logic in `PythonPipProvider.getPythonController()`:
- If both `TRUSTIFY_DA_PIP_SHOW` and `TRUSTIFY_DA_PIP_FREEZE` env vars are set, use RealEnv with generic python/pip (test injection mode)
- Otherwise, detect python3/pip3 (falling back to python/pip), and use VirtualEnv if `TRUSTIFY_DA_PYTHON_VIRTUAL_ENV=true`

## Edge Cases NOT Handled

- `-r` / `--requirement` includes (recursive requirements files)
- `-e` / `--editable` installs
- `--hash` verification
- `-c` / `--constraint` files
- `-f` / `--find-links`
- `-i` / `--index-url` or `--extra-index-url`
- VCS URLs (e.g., `git+https://...`)
- Lines with `@ file` in pip freeze output are filtered out when building the dep cache, but no similar handling exists in requirements.txt parsing

## Design Decisions

1. **Why delegate to pip for resolution?** The requirements.txt format only declares direct dependencies (often pinned). Transitive dependency resolution requires pip's solver or the installed environment. The client requires packages to be installed before analysis.

2. **Why HashMap with StringInsensitive?** Python package names are case-insensitive per PEP 426, and pip normalizes `-`/`_` interchangeably. The triple-key storage handles the most common normalization cases.

3. **Why two SBOM modes?** Stack analysis includes the full transitive tree (for deeper vulnerability analysis), while component analysis provides only direct dependencies (faster, used for IDE inline hints).

4. **Why re-read manifest for ignore detection?** The ignore pattern scan operates on raw manifest text (needs inline comments), while dependency resolution operates on the filtered/parsed dependency list. They are separate concerns processed at different stages.
