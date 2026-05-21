# Cargo Workspace License Inheritance Fix

**Feature:** TC-4528 — Fix 400 CycloneDX validation error for `license = { workspace = true }` in Cargo.toml
**PRs:** Java PR#486, JS PR#532
**Date:** 2026-05-20

## Problem

When a Cargo.toml uses workspace license inheritance (`license = { workspace = true }`),
TOML parsers return a non-string value that flows into the CycloneDX SBOM as an invalid
license object, causing a 400 validation error from the backend.

- **JS:** `smol-toml` returns `{ workspace: true }` — a truthy JS object that
  short-circuits the `||` chain and gets embedded in the SBOM
- **Java:** `tomlj`'s `getString("package.license")` returns `null` for inline tables,
  but `get("package.license")` returns a `TomlInlineTable` — must use `instanceof String`

## Implementation Pattern

### Java (`CargoProvider.java`)

Replicates the existing workspace version resolution pattern from `parseCargoToml()`:

```java
Object licenseValue = tomlResult.get(PACKAGE_LICENSE);
if (licenseValue instanceof String) {
    license = (String) licenseValue;
} else if (licenseValue != null) {
    Boolean isWorkspaceLicense = tomlResult.getBoolean(PACKAGE_LICENSE_WORKSPACE);
    if (Boolean.TRUE.equals(isWorkspaceLicense)) {
        license = tomlResult.getString(WORKSPACE_PACKAGE_LICENSE);
    }
}
```

Resolution is done within the same Cargo.toml file (which contains both `[package]`
and `[workspace.package]` sections in the test fixture). Real-world workspace
inheritance across files requires `cargo metadata` (future enhancement).

### JavaScript (`rust_cargo.js`)

Two-part fix:
1. `readLicenseFromManifest(manifest, metadata)` — detects `{ workspace: true }` and
   calls `resolveWorkspaceLicense(metadata)` which reads the workspace root Cargo.toml
   using the `workspace_root` field from `cargo metadata`
2. Defensive guard in `cyclone_dx_sbom.js` `getComponent()` — filters out non-CycloneDX
   license objects (anything without a `license` or `expression` key)

## Test Coverage

- Both clients: workspace inheritance resolves license from `[workspace.package]`
- Both clients: graceful null when workspace section has no license
- JS only: defensive SBOM guard filters `{ workspace: true }` objects
- JS only: full SBOM integration test with workspace license in expected output

## Parity Notes

Both implementations are functionally aligned. The JS client has slightly deeper
workspace support (reads workspace root via `cargo metadata.workspace_root`), while
the Java client resolves within the same TOML file. Both produce the same result
for the common case where `[workspace.package]` is in the same file.
