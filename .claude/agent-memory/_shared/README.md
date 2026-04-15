# Shared Agent Knowledge

This directory is used by agents to share cross-cutting knowledge with each other.
Agents cannot communicate directly, so they read and write files here as a coordination mechanism.

## What belongs here

This directory holds **living knowledge** that agents actively reference. Two categories:

### Persistent (stays until superseded)
- `parity-matrix.md` — Running feature parity status (maintained by da-parity)
- `parity-report-*.md` — Detailed parity analysis snapshots (supersede older reports on the same scope)
- `spec-<feature>.md` — Language-neutral implementation specs (written by da-parity)
- `impl-<project>-<feature>.md` — Implementation notes (written by client/IDE agents after implementing)
- `contract-<topic>.md` — API contract notes (written by api-contract-review)

### Issue investigations

Investigation findings for bugs and issues belong on the issue itself (Jira comment or GitHub comment), NOT as files in this directory. This keeps the repo clean and makes findings visible to the whole team.

**Guardrail:** Agents must NEVER post comments to Jira or GitHub without explicit user consent. The workflow is: investigate → present findings to the user → ask permission before posting.

## Workflow
1. **Before implementation:** da-parity writes a `spec-<feature>.md` with language-neutral requirements
2. **During implementation:** Client agents read the spec and reference the other client's code
3. **After implementation:** Client agents write `impl-<project>-<feature>.md` with notes about their approach
4. **Verification:** da-parity reads both implementations and writes a `parity-report-<date>.md`

## Rules
- Keep files concise and actionable
- Include file paths and function names for easy reference
- Update or remove stale files — don't let the directory grow unbounded
- Specs should be language-neutral (describe WHAT, not HOW in a specific language)
- Parity reports stay until superseded by a newer report covering the same scope
