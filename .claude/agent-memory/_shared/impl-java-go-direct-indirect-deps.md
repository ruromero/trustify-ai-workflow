# Go Provider: Direct vs Indirect Dependency Filtering (TC-4300)

## What was fixed
The Java client's `GoModulesProvider` now correctly distinguishes direct from indirect dependencies in Go 1.17+ modules, achieving parity with the JS client's `golang_gomodules.js`.

## How it works
Both clients now follow the same pattern:

1. Run `go mod edit -json` to get structured go.mod data
2. Parse the `Require` array, filtering entries where `Indirect` is `false` or absent
3. Build a set of direct dependency module paths
4. When processing `go mod graph` edges:
   - For root-level edges: only include deps whose module path is in the direct set
   - For non-root edges: include all deps (these are transitive dependency relationships)

## Java implementation details
- New method `getDirectDependencyPaths(Path)` runs `go mod edit -json` and returns `Set<String>` of direct dep paths
- New helper `getModulePath(String)` strips `@version` suffix from graph entries for comparison
- Both `buildSbomFromGraph` (stack analysis) and `buildSbomFromList` (component analysis) accept and use the direct dep set
- Important: root comparison in `buildSbomFromGraph` uses `getModulePath(key).equals(getModulePath(rootPackage))` because MVS processing adds version suffix to root key

## JS client reference
The JS client already had this implemented. The key code is in `getSBOM()` in `src/providers/golang_gomodules.js`, using `getPackageName()` (equivalent to Java's `getModulePath`) and `directDepPaths` Set.

## Files changed
- `src/main/java/io/github/guacsec/trustifyda/providers/GoModulesProvider.java`
- Test fixtures in `src/test/resources/tst_manifests/golang/` and `src/test/resources/msc/golang/mvs_logic/`
