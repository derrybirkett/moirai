# Changelog

All notable changes to Moirai.

The format is based on [Keep a Changelog](https://keepachangelog.com/), and Moirai adheres to [Semantic Versioning](https://semver.org/). Consuming repos pin a tag (e.g. `v0.1.0`) so the orchestrator's behaviour is stable across consumer updates.

## [0.3.1] — 2026-05-28

Registry-only patch: bumps Auditor pin to `v0.3.0` and flips its default sink from `inbox` to `issues`. No adapter or installer changes — the `issues` sink path was already supported via Curator's `schedule + issues` combo.

### Changed
- `registry.yml`: `auditor.version` `v0.2.0` → `v0.3.0`; `auditor.default_sink` `inbox` → `issues`. Description updated.

### Why

Auditor `v0.3.0` switched from writing findings to `notes/inbox.md` to filing GitHub Issues. Driven by a concrete bug: the nightly auditor's commit to `inbox.md` on main caused a merge conflict that blocked an unrelated PR merge. More fundamentally, agent-generated findings belong in the triage queue (where Curator already files) rather than injected into the developer's session inbox.

### Migration for existing consumers

After bumping `.moirai` to `v0.3.1` and `.agents/auditor` to `v0.3.0`:

```bash
.moirai/bin/install auditor --sink issues --force
```

This regenerates the workflow with `issues: write` permission (instead of `contents: write`) and drops the inbox commit-back step.

## [0.3.0] — 2026-05-27

Third sink+trigger combo: `schedule` + `inbox`. Enables nightshift agents that batch-audit open PRs and commit back to `main` instead of per-PR branches. Auditor `v0.2.0` is the first agent to use this combo.

### Why

The per-PR auditor (introduced in `v0.2.0`) became a blocker in practice: the auditor-bot commit-back to PR branches caused rebase friction, on-every-push timing slowed PR cycles, and per-PR API costs scaled with team activity. Coherence findings are tidiness work — they suit a nightshift cadence.

### Added
- New conditional block `{{IF_NOT_PR}}…{{ENDIF_NOT_PR}}` in the adapter template — keeps content when the trigger is non-PR (schedule, workflow_dispatch). Stripped otherwise.
- The inbox commit-back step in `adapters/claude-code-action/workflow.yml.template` now branches:
  - **PR mode** (existing): commit message `chore(<agent>): refresh PR #N findings`; push to PR head ref.
  - **Non-PR mode** (new): commit message `chore(<agent>): nightly findings — YYYY-MM-DD`; push to the workflow's own ref (`${{ github.ref_name }}`).
- `bin/install` now processes three conditional passes: `INBOX` first (outer), then `PR` and `NOT_PR` (inner). Nesting works because outer-then-inner ordering preserves the inner markers when the outer block is kept.
- `protocol/SPEC.md` §4 documents the trigger-dependent push target. The fork-safety paragraph clarifies that fork safety only applies to PR triggers; schedule triggers commit to the consuming repo's own ref, so fork PRs are safe to audit.

### Changed
- `registry.yml`: `auditor` default trigger flipped from `pull_request` → `schedule`. Pin bumped to `v0.2.0` (the agent now supports both modes via PROMPT-level mode detection).

### Migration
- Consumers of auditor `v0.1.0` on Moirai `v0.2.0` keep working — nothing changes for them. To pick up the nightshift default, bump `.moirai` to `v0.3.0` AND `.agents/auditor` to `v0.2.0`, then re-run `.moirai/bin/install auditor --force` to regenerate the workflow with the new trigger.
- Curator consumers are unaffected — the schedule+issues path is unchanged.

## [0.2.0] — 2026-05-27

Second-agent release — the protocol now supports the `pull_request` trigger and the `inbox` sink end-to-end, validated by extracting [Auditor](https://github.com/derrybirkett/auditor) as the second agent (after [Curator](https://github.com/derrybirkett/curator)). This was step 4 of the walking-skeleton plan.

### Added
- [`registry.yml`](registry.yml) — `auditor` entry, pinned to `v0.1.0`. Default trigger `pull_request`, default sink `inbox`.
- [`adapters/claude-code-action/workflow.yml.template`](adapters/claude-code-action/workflow.yml.template) now supports the full `pull_request`/`inbox` shape via conditional blocks:
  - PR context resolution step (handles both `pull_request` event and `workflow_dispatch` with a `pr_number` input).
  - Fork-safety skip step (no push permissions on fork branches).
  - `PR_NUMBER`, `PR_BASE_SHA`, `PR_HEAD_SHA` injected into the runtime envelope.
  - Commit-back step that diffs the inbox path, commits as `<agent>-bot`, and pushes to the PR head ref.
- `--force` flag on [`bin/install`](bin/install) to overwrite an existing `config.yml` / workflow file. Default behaviour (leave in place) is unchanged.
- The scaffolder now strips `defaults.yml`'s own leading comment block before concatenating its own header — fixes the stacked-header cosmetic noise from `v0.1.x`.
- [`protocol/SPEC.md`](protocol/SPEC.md) §3 now documents the trigger-specific envelope (`PR_NUMBER`, `PR_BASE_SHA`, `PR_HEAD_SHA` for `pull_request` triggers).
- [`protocol/SPEC.md`](protocol/SPEC.md) §4 now documents the `inbox` sink commit-back contract and the fork-safety requirement for adapters.

### Changed
- Template substitution moved from one-shot `dict.replace` to a small Python pass that first strips/keeps `{{IF_PR}}…{{ENDIF_PR}}` and `{{IF_INBOX}}…{{ENDIF_INBOX}}` blocks based on trigger and sink, then substitutes scalar placeholders. Backward-compatible for the `schedule`/`issues` path (Curator's generated workflow is the same shape modulo cosmetic tidy).

### Migration

Consuming repos pinned at `v0.1.x` can stay there — the schedule+issues path is functionally unchanged. To pick up the new `pull_request`/`inbox` support, bump the `.moirai` submodule to `v0.2.0` and run `bin/install <new-agent>` for any newly-registered agents.

## [0.1.2] — 2026-05-27

Fixes the first install-script bug surfaced by step 3 of the walking skeleton (consume curator via moirai in monospace.studio).

### Fixed
- [`bin/install`](bin/install) now places agent submodules at `.agents/<name>/` (top-level, sibling to `.moirai/`) instead of `.moirai/agents/<name>/`. Git rejects nested submodules, which broke the first install attempt with `fatal: Pathspec '.moirai/agents/curator' is in submodule '.moirai'`.
- [`protocol/SPEC.md`](protocol/SPEC.md) §2 updated to document the new `.agents/<name>/` convention.

## [0.1.1] — 2026-05-27

Fixes the missing-auth issue surfaced by Curator's first live nightly run on monospace.studio: the run succeeded but filed zero issues because Claude's Bash sub-shells had no GitHub auth and the `mcp__github__*` tools listed in the old workflow's allowed-tools weren't backed by an actual MCP server.

### Fixed
- [`adapters/claude-code-action/workflow.yml.template`](adapters/claude-code-action/workflow.yml.template) now surfaces `GH_TOKEN` and `GITHUB_TOKEN` as environment variables in the run step, so agents that use the `issues` sink (or otherwise need `gh` via Bash) can authenticate without extra wiring. The default `ALLOWED_TOOLS` (`Bash,Read,Grep,Glob`) was already free of phantom MCP tools, so no change there.

## [0.1.0] — 2026-05-27

Walking-skeleton release. The protocol works end-to-end for one agent ([curator](https://github.com/derrybirkett/curator)).

### Added
- [`protocol/SPEC.md`](protocol/SPEC.md) — the contract every agent and adapter must follow: required files in an agent repo, config shape rules, environment-variable envelope, sink interface, adapter responsibilities, versioning.
- [`registry.yml`](registry.yml) — the catalog. Curator is the first entry, pinned to `v0.1.0`. Adapter manifest declares `claude-code-action` as the only adapter for now.
- [`adapters/claude-code-action/`](adapters/claude-code-action/) — GitHub Actions adapter wrapping `anthropics/claude-code-action@v1`. README + parameterised `workflow.yml.template`.
- [`bin/install`](bin/install) — bash installer. `moirai/bin/install <agent>` submodules the agent at its pinned tag, scaffolds `.github/agents/<name>/config.yml` from the agent's `defaults.yml`, and drops a workflow from the chosen adapter template. Supports `--trigger`, `--sink`, `--adapter` overrides.
- `README.md`, `LICENSE` (MIT).

### Known limitations
- One adapter (`claude-code-action`). OpenAI Assistants / local CLI adapters are anticipated but not specified.
- One agent in the registry (`curator`). [Auditor](https://github.com/derrybirkett/monospace.studio/blob/main/.github/agents/auditor/PROMPT.md) is in-repo at monospace.studio and will be extracted next, validating that the protocol accommodates a second agent.
- No `--force` flag on `bin/install` yet; existing config and workflow files are left in place.
- The minimal YAML reader in `bin/install` is shape-specific to `registry.yml`. Reaches for `python3` if available, with an awk fallback. Will graduate to a proper parser when there's a second adapter that needs it.
