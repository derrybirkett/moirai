# Changelog

All notable changes to Moirai.

The format is based on [Keep a Changelog](https://keepachangelog.com/), and Moirai adheres to [Semantic Versioning](https://semver.org/). Consuming repos pin a tag (e.g. `v0.1.0`) so the orchestrator's behaviour is stable across consumer updates.

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
