# Agent Architecture

This document describes the specialized AI subagent architecture for the Trusted Supply Chain projects. All agents are defined in `.claude/agents/` and are available in every session started from this repository.

## Overview

The architecture uses 15 specialized agents organized by project domain. Three layers separate concerns cleanly:

| Layer | Responsibility | Location |
|---|---|---|
| **Agent** | Domain logic, module ownership, architecture patterns, how components interact | `.claude/agents/<name>.md` |
| **CONVENTIONS.md** | Code style, naming rules, build/format commands, test patterns | Each repository's root |
| **Skills (sdlc-workflow)** | Generic process steps — how to implement a task, verify a PR, plan a feature | Plugin skills |

Agents contain **what the code does and how it fits together**. They do not repeat build commands or style rules — those belong in each repo's `CONVENTIONS.md`. Skills describe **how to perform a workflow** generically; they discover conventions from `CONVENTIONS.md` at runtime.

Agents cannot spawn other agents. The main conversation thread orchestrates them — invoking them sequentially or in parallel as needed. When a skill like `/implement-task` targets a specific repository, the main thread delegates to the appropriate domain agent. Cross-project coordination happens through shared knowledge files in `.claude/agent-memory/_shared/`.

## Agent Inventory

### Trustify Backend (Rust) — 4 Agents

The Trustify backend is a modular Rust monolith. It is split into four specialist agents by subsystem.

#### trustify-api

| Field | Value |
|---|---|
| **Focus** | REST endpoints, query DSL, OpenAPI generation, fundamental/analysis modules |
| **Codebase** | `modules/fundamental/`, `modules/analysis/`, `server/`, `query/`, `trustd/`, `openapi.yaml` |
| **Serena** | serena-trustify |
| **Model** | inherit |
| **Hooks** | `cargo fmt` on edit |
| **Skills** | sdlc-workflow:define-feature, plan-feature, implement-task, verify-pr |

Key knowledge: endpoint pattern (`/v2/<resource>`), utoipa annotations, `Require<Permission>` extractors, `Graph` abstraction for DB queries, `thiserror` + `actix_web::ResponseError` for error handling.

#### trustify-ingestion

| Field | Value |
|---|---|
| **Focus** | Data ingestion pipeline, SBOM/CSAF/CVE parsing, graph construction, scheduled importers |
| **Codebase** | `modules/ingestor/`, `modules/importer/` |
| **Serena** | serena-trustify |
| **Model** | inherit |
| **Hooks** | `cargo fmt` on edit |
| **Skills** | sdlc-workflow:define-feature, plan-feature, implement-task, verify-pr |

Key knowledge: `IngestorService` orchestration, graph node construction, `ImportRunner` with source-specific handlers (SBOM, CSAF, CVE, OSV, ClearlyDefined, Quay, CWE), `sbom-walker`/`csaf-walker` libraries, transactional ingestion.

#### trustify-data

| Field | Value |
|---|---|
| **Focus** | Database entities, SeaORM models, migrations, storage backends |
| **Codebase** | `entity/` (48 tables), `migration/`, `modules/storage/`, `common/db/`, `test-context/` |
| **Serena** | serena-trustify |
| **Model** | inherit |
| **Hooks** | `cargo fmt` on edit |
| **Skills** | sdlc-workflow:define-feature, plan-feature, implement-task, verify-pr |

Key knowledge: SeaORM `DeriveEntityModel`, junction tables for many-to-many, `DispatchBackend` trait (S3, filesystem, temp), zstd compression, `TrustifyContext` test harness, idempotent migrations.

#### trustify-infra

| Field | Value |
|---|---|
| **Focus** | Authentication, OIDC, authorization, permissions, observability, HTTP server, deployment |
| **Codebase** | `common/auth/`, `common/infrastructure/`, `common/`, `modules/user/`, `modules/ui/`, `etc/deploy/` |
| **Serena** | serena-trustify |
| **Model** | inherit |
| **Hooks** | `cargo fmt` on edit |
| **Skills** | sdlc-workflow:define-feature, plan-feature, implement-task, verify-pr |

Key knowledge: OIDC provider integration (Keycloak, garage-door embedded), JWT validation, RBAC permission model, OpenTelemetry tracing/metrics, Actix-web server builder, deployment profiles (API, Importer, PM mode).

### Trustify UI (React/TypeScript) — 1 Agent

#### trustify-ui

| Field | Value |
|---|---|
| **Focus** | React frontend: pages, components, PatternFly 6, TanStack Query hooks, table controls, API client generation, OIDC auth |
| **Codebase** | `client/` (React app), `server/` (Express proxy), `common/`, `e2e/` (Playwright tests), `branding/` |
| **Serena** | serena-trustify-ui |
| **Model** | inherit |
| **Hooks** | `biome check` on edit |
| **Skills** | sdlc-workflow:define-feature, plan-feature, implement-task, verify-pr |
| **Extra MCP** | Playwright (browser testing), Context7 (library docs) |

Key knowledge: page pattern (shell + context + toolbar + table), TanStack Query v5 hooks, 36+ table-control hooks with URL state persistence, auto-generated API client via `@hey-api/openapi-ts` (never edit manually), PatternFly 6 components, Rsbuild, OIDC via `react-oidc-context`.

### Dependency Analytics — 7 Agents

#### da-backend

| Field | Value |
|---|---|
| **Focus** | Trustify Dependency Analytics Java/Quarkus backend: Camel routes, vulnerability providers, SBOM parsing, license analysis, report generation |
| **Codebase** | `io.github.guacsec.trustifyda` — `integration/backend/`, `integration/providers/`, `integration/sbom/`, `integration/licenses/`, `integration/report/` |
| **Serena** | serena-trustify-dependency-analytics |
| **Model** | inherit |
| **Hooks** | `mvn spotless:apply` on edit |
| **Skills** | sdlc-workflow:define-feature, plan-feature, implement-task, verify-pr |
| **Extra MCP** | Context7 |

Key knowledge: Camel routing (direct/seda/multicast), provider pattern with `ProviderResponseHandler`, Trustify integration (`/api/v2/vulnerability/analyze`, `/api/v2/purl/recommend`), Redis caching, Freemarker HTML reports, CycloneDX/SPDX SBOM parsing via `SbomParserFactory`.

#### da-api-spec

| Field | Value |
|---|---|
| **Focus** | OpenAPI 3.0.3 specification, API model evolution, code generation, backward compatibility |
| **Codebase** | `api/v5/openapi.yaml`, `pom.xml` |
| **Serena** | serena-trustify-da-api-spec |
| **Model** | sonnet |
| **Skills** | sdlc-workflow:define-feature, plan-feature, implement-task, verify-pr |

Key knowledge: API version 5.1.0, endpoints (`/api/v5/analysis`, `/api/v5/batch-analysis`, `/api/v5/licenses/*`), generates Java model classes (`trustify-da-api-model` Maven artifact) and TypeScript types consumed by all downstream clients. Changes here affect the entire DA ecosystem.

#### da-java-client

| Field | Value |
|---|---|
| **Focus** | Trustify Dependency Analytics Java client library: ecosystem providers (Maven, Gradle, pip, Go, Cargo, npm), SBOM generation, HTTP client |
| **Codebase** | `io.github.guacsec.trustifyda` — `impl/`, `providers/`, `sbom/`, `license/`, `image/`, `tools/` |
| **Serena** | serena-trustify-da-java-client + serena-trustify-da-javascript-client (cross-reference) |
| **Model** | inherit |
| **Hooks** | `mvn spotless:apply` on edit |
| **Skills** | sdlc-workflow:define-feature, plan-feature, implement-task, verify-pr |
| **Extra MCP** | Context7 |

Key knowledge: provider interface pattern, `ExhortApi` HTTP client (~785 lines), `Ecosystem` enum (8 ecosystems), `CycloneDXSbom` generation, centralized `IgnorePatternDetector` for both `exhortignore` and `trustify-da-ignore` markers, Java 21 + Maven.

**Must maintain functional parity with `da-js-client`.** Cross-references JS client code via Serena and reads/writes shared knowledge in `.claude/agent-memory/_shared/`.

#### da-js-client

| Field | Value |
|---|---|
| **Focus** | JavaScript/TypeScript client library: ecosystem providers, tree-sitter parsing, OCI image analysis, workspace discovery |
| **Codebase** | `src/` — `index.js`, `analysis.js`, `provider.js`, `providers/` (24 files), `sbom.js`, `license/`, `oci_image/`, `workspace.js` |
| **Serena** | serena-trustify-da-javascript-client + serena-trustify-da-java-client (cross-reference) |
| **Model** | inherit |
| **Hooks** | `biome check` on edit |
| **Skills** | sdlc-workflow:define-feature, plan-feature, implement-task, verify-pr |
| **Extra MCP** | Context7 |

Key knowledge: provider chain with priority-ordered matching (Poetry -> uv -> pip fallback for pyproject.toml), tree-sitter WASM for Go mod and requirements.txt parsing, `workspace.js` for monorepo discovery, `stackAnalysisBatch()` for multi-manifest analysis, Node.js 20+.

**Must maintain functional parity with `da-java-client`.** Cross-references Java client code via Serena and reads/writes shared knowledge in `.claude/agent-memory/_shared/`.

#### da-intellij

| Field | Value |
|---|---|
| **Focus** | IntelliJ IDEA plugin: inspections, annotators, intention actions, PSI parsing, settings |
| **Codebase** | `org.jboss.tools.intellij` — `settings/`, `stackanalysis/`, `componentanalysis/` (with `pypi/`, `golang/` subpackages) |
| **Serena** | serena-intellij-dependency-analytics + serena-fabric8-analytics-vscode-extension + serena-trustify-da-java-client (cross-references) |
| **Model** | inherit |
| **Hooks** | `./gradlew ktlintFormat` on edit |
| **Skills** | sdlc-workflow:define-feature, plan-feature, implement-task, verify-pr |
| **Extra MCP** | Context7 |

Key knowledge: IntelliJ Platform SDK, PSI (Program Structure Interface) for requirements.txt, Inspections/Annotators/IntentionActions, `plugin.xml` registration, Gradle with Kotlin DSL, depends on `trustify-da-api-model`.

**Must maintain UX parity with `da-vscode`.** Checks Java client capabilities and reads shared knowledge.

#### da-vscode

| Field | Value |
|---|---|
| **Focus** | VS Code extension: diagnostics, code actions, webview panels, OIDC authentication, LLM analysis |
| **Codebase** | `src/` — `extension.ts`, `exhortServices.ts`, `providers/`, `authentication/`, `diagnosticsPipeline.ts`, `codeActionHandler.ts`, `dependencyReportPanel.ts` |
| **Serena** | serena-fabric8-analytics-vscode-extension + serena-intellij-dependency-analytics + serena-trustify-da-javascript-client (cross-references) |
| **Model** | inherit |
| **Hooks** | `biome check` on edit |
| **Skills** | sdlc-workflow:define-feature, plan-feature, implement-task, verify-pr |
| **Extra MCP** | Context7 |

Key knowledge: VS Code Extension API, `onDidOpenTextDocument` activation, `DiagnosticCollection` for inline warnings, `CodeActionProvider` for quick fixes, webview panels with Handlebars templates, OIDC via `oidcAuthentication.ts`, depends on `@trustify-da/trustify-da-javascript-client`.

**Must maintain UX parity with `da-intellij`.** Checks JS client capabilities and reads shared knowledge.

#### da-parity

| Field | Value |
|---|---|
| **Focus** | Cross-client feature parity: Java↔JS clients, IntelliJ↔VS Code extensions |
| **Tools** | Read-only (Read, Grep, Glob, Bash) |
| **Serena** | All 5 DA instances (both clients, both IDEs, API spec) |
| **Model** | inherit |

This is a **read-only analyst**. It does not modify code. Its responsibilities:

1. **Before implementation** — Analyze how similar features work in both clients, generate language-neutral implementation specs
2. **After implementation** — Compare both sides, report functional differences with file paths and line numbers
3. **Ongoing** — Maintain the parity matrix at `.claude/agent-memory/_shared/parity-matrix.md`

Writes all findings to `.claude/agent-memory/_shared/` where other agents read them.

### Testing — 2 Agents

#### da-integration-tests

| Field | Value |
|---|---|
| **Focus** | Integration test scenarios, spec.yaml fixtures, Python test orchestration, Tekton CI pipelines |
| **Codebase** | `scenarios/` (per-ecosystem test projects), `tasks/` (Tekton), `pipelines/`, `shared-scripts/` (Python) |
| **Model** | sonnet |
| **Skills** | sdlc-workflow:define-feature, plan-feature, implement-task, verify-pr |

Key knowledge: `spec.yaml` format (expected dependency counts, provider sources, license providers), Python test runner (`run_tests.py`), coverage across all 8 ecosystems.

#### trustify-perf

| Field | Value |
|---|---|
| **Focus** | Goose load tests, performance scenarios, OIDC test flows, data replication tools |
| **Codebase** | `src/` — `main.rs`, `oidc.rs`, `db.rs`, `restapi/` (per-endpoint scenarios), `scenarios/`, `replicator/` |
| **Serena** | serena-trustify-scale-testing |
| **Model** | sonnet |
| **Hooks** | `cargo fmt` on edit |
| **Skills** | sdlc-workflow:define-feature, plan-feature, implement-task, verify-pr |

Key knowledge: Goose 0.18 framework, OIDC token acquisition, scenario auto-generation from database (`GENERATE_SCENARIO=true`), JSON5 scenario definitions, humantime format for timeouts.

### Cross-Cutting — 1 Agent

#### api-contract-review

| Field | Value |
|---|---|
| **Focus** | Spec-to-implementation compliance, backward compatibility, cross-service contract auditing |
| **Tools** | Read-only (Read, Grep, Glob, Bash) |
| **Serena** | All 5 relevant instances (API spec, backend, both clients, Trustify) |
| **Model** | sonnet |

This is a **read-only analyst**. It monitors two API contracts:
1. **DA API** (trustify-da-api-spec) — verified against the Trustify Dependency Analytics backend and both client libraries
2. **Trustify REST API** (generated from utoipa) — verified against Exhort backend (consumer) and Trustify UI (consumer)

Reports issues with severity levels: BREAKING, MISMATCH, WARNING.

## Cross-Project Coordination

### Parity Protocol

The DA ecosystem has paired repositories that must maintain functional parity:
- **Client libraries:** trustify-da-java-client ↔ trustify-da-javascript-client
- **IDE extensions:** intellij-dependency-analytics ↔ fabric8-analytics-vscode-extension

The workflow for any feature that spans paired repositories:

```
1. da-parity analyzes existing patterns across both sides
   └── writes spec to _shared/spec-<feature>.md

2. First client agent implements (e.g., da-js-client)
   ├── reads spec from _shared/
   ├── references counterpart via Serena (read-only)
   └── writes notes to _shared/impl-<project>-<feature>.md

3. Second client agent implements (e.g., da-java-client)
   ├── reads spec + first agent's notes from _shared/
   └── writes notes to _shared/impl-<project>-<feature>.md

4. da-parity verifies both implementations
   └── updates _shared/parity-matrix.md
```

### Shared Knowledge Directory

`.claude/agent-memory/_shared/` is the coordination mechanism between agents. It holds **living knowledge** that agents actively reference.

**Persistent files** (stay until superseded):

| Pattern | Purpose | Written By |
|---|---|---|
| `spec-<feature>.md` | Language-neutral implementation specs | da-parity |
| `impl-<project>-<feature>.md` | Implementation notes after completing work | Client/IDE agents |
| `parity-matrix.md` | Running feature parity status | da-parity |
| `parity-report-*.md` | Detailed parity analysis reports | da-parity |
| `contract-<topic>.md` | API contract compliance notes | api-contract-review |

**Issue investigations** go on the issue itself (Jira/GitHub comment), not as files in this directory. This keeps the repo clean and makes findings visible to the whole team. Agents must NEVER post comments without explicit user consent.

Any agent that discovers cross-cutting information during investigations, bug fixes, or code reviews updates the relevant shared docs. This keeps the knowledge current organically.

### Agent Memory

Each agent has `memory: project` which creates a persistent directory at `.claude/agent-memory/<agent-name>/`. This memory:
- Survives across sessions
- Focuses on **current implementation understanding and architecture decisions** — how things work and why they were built that way
- Is private to each agent (unlike `_shared/` which is cross-agent)

Bugs, gaps, and parity issues go to `.claude/agent-memory/_shared/` where all relevant agents can see them.

### Information Flow

```
                        CLAUDE.md conventions
                               │
                        ┌──────┴──────┐
                        │ Main thread │
                        │(orchestrator)│
                        └──────┬──────┘
                               │
          ┌────────────────────┼────────────────────────┐
          │                    │                         │
     ┌────┴────┐         ┌────┴────┐            ┌──────┴───────┐
     │da-parity│◄───────►│ _shared/│◄──────────►│api-contract  │
     │(analyze)│  writes  │  docs   │   writes   │  (review)    │
     └────┬────┘         └────┬────┘            └──────────────┘
          │                   │ reads
     ┌────┼───────────────────┼──────────────────┐
     │    │    │    │         │    │              │
   Java  JS  IntelliJ  VS Code  Backend     API Spec
  client client plugin  ext.   (Trustify Dependency Analytics)     (OpenAPI)
```

## MCP Servers

### Serena Instances (Code Intelligence)

Each Serena instance provides semantic code navigation for one repository: symbol lookup, reference finding, symbol body reading/editing, pattern search.

| Instance | Repository |
|---|---|
| serena-trustify | trustify (Rust backend) |
| serena-trustify-ui | trustify-ui (React frontend) |
| serena-trustify-dependency-analytics | Trustify Dependency Analytics (Java backend) |
| serena-trustify-da-api-spec | trustify-da-api-spec (OpenAPI) |
| serena-trustify-da-java-client | trustify-da-java-client (Java client) |
| serena-trustify-da-javascript-client | trustify-da-javascript-client (JS client) |
| serena-intellij-dependency-analytics | intellij-dependency-analytics (IntelliJ plugin) |
| serena-fabric8-analytics-vscode-extension | fabric8-analytics-vscode-extension (VS Code) |
| serena-trustify-scale-testing | trustify-scale-testing (load tests) |

### Additional MCP Servers

| Server | Scoped To | Purpose |
|---|---|---|
| Context7 | trustify-ui, da-backend, da-java-client, da-js-client, da-intellij, da-vscode | Up-to-date library documentation (PatternFly, Quarkus, SeaORM, etc.) |
| Playwright | trustify-ui | Browser automation for UI testing |
| GitHub | Main conversation | PR/issue management |

## Quality Enforcement

Quality enforcement is split across the three layers:

### Formatting Hooks (automatic)

Each agent has a `PostToolUse` hook that runs the appropriate formatter after every edit:

| Language | Formatter | Agents |
|---|---|---|
| Rust | `cargo fmt` | trustify-api, trustify-ingestion, trustify-data, trustify-infra, trustify-perf |
| Java | `mvn spotless:apply` | da-backend, da-java-client |
| TypeScript/JavaScript | `biome check --write` | trustify-ui, da-js-client, da-vscode |
| Kotlin | `./gradlew ktlintFormat` | da-intellij |

### Build/Test/Lint (CONVENTIONS.md)

Full build, test, and CI check commands live in each repository's `CONVENTIONS.md`. The `implement-task` skill discovers these automatically (Step 4) and runs them during self-verification (Step 9).

### Domain Rules (agent descriptions)

Each agent contains domain-specific rules that go beyond style — invariants, compatibility requirements, and architectural constraints. For example: "migrations must be idempotent", "provider behavior must match the counterpart exactly", "never edit auto-generated files".
