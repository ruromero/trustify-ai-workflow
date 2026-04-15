# Trustify AI Workflow

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) workflow for developing across the Trustify and Dependency Analytics ecosystem. It provides 15 specialized AI agents that understand the domain, architecture, and cross-project relationships of each repository.

## What This Gives You

- **Specialized agents** for each subsystem (Rust backend, React UI, Java/JS client libraries, IntelliJ/VS Code extensions, API spec, backend service, load tests)
- **Cross-project coordination** with automatic parity checks between paired repositories (Java client ↔ JS client, IntelliJ ↔ VS Code)
- **SDLC workflow** via the [sdlc-workflow plugin](https://github.com/mrizzi/sdlc-plugins) for feature definition, planning, implementation, and PR verification — all integrated with Jira

## Quick Start

1. Set the environment variable pointing to your workspace:

   ```sh
   export TRUSTIFY_WS_DIR=/path/to/your/workspace
   ```

2. Clone the target repositories under `$TRUSTIFY_WS_DIR` (see [usage guide](docs/usage-guide.md#1-clone-the-repositories) for the expected layout)

3. Install the [sdlc-workflow plugin](https://github.com/mrizzi/sdlc-plugins) and create your local `.claude/plugins.json` and `.claude/settings.local.json`

4. Start Claude Code from this repository:

   ```sh
   claude
   ```

See the [Usage Guide](docs/usage-guide.md) for full setup instructions.

## Documentation

- [Usage Guide](docs/usage-guide.md) — Setup, workflow, and how to work with agents
- [Agent Architecture](docs/agent-architecture.md) — Agent inventory, cross-project coordination, and shared knowledge

## Repository Structure

```
.claude/
  agents/              # 15 specialized agent definitions
  agent-memory/
    _shared/           # Cross-agent knowledge (parity matrix, specs, reports)
    <agent-name>/      # Per-agent implementation notes
  settings.json        # Shared settings (tracked)
  settings.local.json  # Personal settings (gitignored)
  plugins.json         # Plugin paths (gitignored)
CLAUDE.md              # Project configuration (repo registry, Jira config, conventions)
docs/
  usage-guide.md       # How to set up and use the workflow
  agent-architecture.md # Agent inventory and coordination details
```

## Agents

| Domain | Agents |
|---|---|
| Trustify Backend (Rust) | `trustify-api`, `trustify-ingestion`, `trustify-data`, `trustify-infra` |
| Trustify UI (React/TS) | `trustify-ui` |
| Dependency Analytics | `da-backend`, `da-api-spec`, `da-java-client`, `da-js-client`, `da-intellij`, `da-vscode` |
| Cross-Cutting | `da-parity`, `api-contract-review` |
| Testing | `da-integration-tests`, `trustify-perf` |

## SDLC Workflow

The [sdlc-workflow plugin](https://github.com/mrizzi/sdlc-plugins) provides four skills:

| Skill | Purpose |
|---|---|
| `/define-feature` | Interactively define a Jira Feature issue |
| `/plan-feature` | Generate implementation plan and Jira tasks from a feature |
| `/implement-task` | Implement a Jira task: write code, run tests, create PR |
| `/verify-pr` | Verify a PR against its Jira task's acceptance criteria |

## License

Apache License 2.0 — see [LICENSE](LICENSE).
