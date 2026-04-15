# Usage Guide

This repository provides a Claude Code AI workflow for the Trusted Supply Chain projects. It includes 15 specialized agents, shared knowledge, and an SDLC plugin that together form a development workflow for multi-repository work.

## Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- The [sdlc-workflow plugin](https://github.com/mrizzi/sdlc-plugins) cloned locally
- All target repositories cloned under a common parent directory

## Setup

### 1. Clone the repositories

All repositories referenced by agents must be cloned under a single parent directory. Set the `TRUSTIFY_WS_DIR` environment variable to this path:

```sh
export TRUSTIFY_WS_DIR=/path/to/your/workspace
```

Expected layout:

```
$TRUSTIFY_WS_DIR/
├── guacsec/
│   ├── trustify-da-java-client/        # Java client library
│   ├── trustify-da-javascript-client/  # JS/TS client library
│   ├── trustify-da-api-spec/           # OpenAPI specification
│   └── trustify-dependency-analytics/  # Java/Quarkus backend (Exhort)
├── redhat-developer/
│   └── intellij-dependency-analytics/  # IntelliJ plugin
├── fabric8-analytics/
│   └── fabric8-analytics-vscode-extension/  # VS Code extension
└── trustification/
    ├── trustify/                        # Rust backend (via Serena)
    ├── trustify-ui/                     # React frontend
    └── trustify-scale-testing/          # Load tests
```

Agent files, hooks, and `CLAUDE.md` all use `${TRUSTIFY_WS_DIR}` so they work across machines without edits.

### 2. Clone the sdlc-workflow plugin

```sh
git clone https://github.com/mrizzi/sdlc-plugins.git $TRUSTIFY_WS_DIR/../sdlc-plugins
```

### 3. Configure local settings

Create `.claude/plugins.json` pointing to your local plugin clone:

```json
{
  "source": "/absolute/path/to/sdlc-plugins/plugins/sdlc-workflow"
}
```

Create `.claude/settings.local.json` with your `additionalDirectories` (absolute paths to repos Claude Code should have access to):

```json
{
  "permissions": {
    "additionalDirectories": [
      "/absolute/path/to/intellij-dependency-analytics",
      "/absolute/path/to/trustify-da-java-client",
      "/absolute/path/to/trustify-da-api-spec",
      "/absolute/path/to/fabric8-analytics-vscode-extension",
      "/absolute/path/to/trustify-da-javascript-client",
      "/absolute/path/to/trustify-dependency-analytics",
      "/absolute/path/to/trustify-scale-testing"
    ]
  }
}
```

Both files are gitignored — they contain machine-specific absolute paths.

### 4. Verify

Start Claude Code from the workflow repository root:

```sh
cd /path/to/trustify-ai-workflow
claude
```

The agents, skills, and shared settings should load automatically.

## Architecture Overview

Three layers separate concerns:

| Layer | What it contains | Where it lives |
|---|---|---|
| **Agent** | Domain logic, module ownership, architecture patterns | `.claude/agents/<name>.md` |
| **CONVENTIONS.md** | Code style, naming rules, build/format commands, test patterns | Each repository's root |
| **Skills (sdlc-workflow)** | Generic process steps — how to perform a workflow | Plugin skills |

Agents know **what the code does**. CONVENTIONS.md knows **how to build and format it**. Skills know **how to run a workflow** (and discover conventions at runtime).

See [agent-architecture.md](agent-architecture.md) for the full agent inventory and cross-project coordination details.

## Development Workflow

The sdlc-workflow plugin provides four skills that map to the software development lifecycle:

### `/define-feature [summary]`

Interactively define a new Jira Feature issue. Walks you through each section of the description template and creates the issue in Jira.

- **Input:** Optional summary text
- **Output:** A Jira Feature issue (e.g., TC-1234)
- **Scope:** Jira only — does not touch code

### `/plan-feature [jira-id] [figma-url]`

Generate an implementation plan from a Jira feature. Analyzes the relevant repositories, produces a plan, and creates Jira tasks.

- **Input:** Jira feature ID and optional Figma URL
- **Output:** Jira tasks with structured descriptions (acceptance criteria, affected files, test requirements)
- **Scope:** Read-only on code, writes to Jira

### `/implement-task [jira-id]`

Implement a Jira task. Reads the task description, identifies the target repository, delegates to the appropriate domain agent, modifies code, runs tests, and creates a PR.

- **Input:** Jira task ID (created by `/plan-feature`)
- **Output:** Code changes, PR, Jira status update
- **Scope:** This is where code gets written

The skill automatically:
1. Reads the task's structured description from Jira
2. Identifies the target repository from the task
3. Discovers build/format/test commands from the repo's `CONVENTIONS.md`
4. Delegates to the domain agent (e.g., `da-java-client` for Java client work)
5. Runs quality checks before creating the PR

### `/verify-pr [jira-id]`

Verify a PR against its Jira task's acceptance criteria. Runs deterministic checks, reads PR review feedback, and creates tracked sub-tasks for required fixes.

- **Input:** Jira task ID that has an associated PR
- **Output:** Verification report posted to GitHub and Jira
- **Scope:** Read-only on code — does not modify or merge

### Typical flow

```
/define-feature "Add Cargo.lock support for workspace resolution"
    │
    ▼
/plan-feature TC-4100
    │  creates tasks: TC-4101, TC-4102, TC-4103
    ▼
/implement-task TC-4101    ← repeat for each task
    │  writes code, creates PR
    ▼
/verify-pr TC-4101
    │  checks PR against acceptance criteria
    ▼
  PR review → merge
```

## Working with Agents Directly

You don't always need the full SDLC workflow. You can ask Claude to delegate to agents directly:

**Investigate a bug:**
> "Investigate why the Go provider drops transitive deps in the Java client"

Claude delegates to `da-java-client`, which uses its Serena instance to navigate the code, cross-checks the JS counterpart, and writes findings to `_shared/` if the issue affects both clients.

**Cross-project analysis:**
> "Compare how Maven dependency exclusions work in both client libraries"

Claude delegates to `da-parity`, which reads both codebases via their Serena instances and writes a comparison to `_shared/`.

**Code review:**
> "Check if the API spec matches what the backend actually returns for /api/v5/analysis"

Claude delegates to `api-contract-review`, which compares the OpenAPI spec against the backend implementation.

**UI work:**
> "Add a filter dropdown for CVE severity on the advisory list page"

Claude delegates to `trustify-ui`, which follows PatternFly 6 patterns and the page structure conventions.

## Shared Knowledge

Agents coordinate through `.claude/agent-memory/_shared/`. Two categories of files:

**Persistent** (stays until superseded):
- `parity-matrix.md` — Feature parity status across paired repositories
- `parity-report-*.md` — Detailed analysis snapshots
- `spec-<feature>.md` — Implementation specs for new features
- `impl-<project>-<feature>.md` — Implementation notes
- `contract-<topic>.md` — API contract compliance notes

**Issue investigations** go on the issue itself (Jira/GitHub comment), not as files in this directory. Agents present findings to the user and offer to post — they never post without explicit consent.

## Agent Memory

Each agent also has a private memory directory at `.claude/agent-memory/<agent-name>/`. This is for **implementation understanding and architecture decisions** — how things work and why they were built that way.

## What's Tracked vs Gitignored

| Tracked (shared) | Gitignored (personal) |
|---|---|
| `CLAUDE.md` | `.env` |
| `.claude/agents/` | `.DS_Store` |
| `.claude/agent-memory/` | `.claude/settings.local.json` |
| `.claude/settings.json` | `.claude/plugins.json` |
| `docs/`, `scripts/` | |
