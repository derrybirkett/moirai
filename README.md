# moirai

The thread that pulls. An orchestrator for thin, independent LLM agents — registry, protocol, installer, and host adapters.

Named after the Greek fates: Clotho spins the thread, Lachesis measures, Atropos cuts. Moirai brings agents into the loom.

## What this is, and what it isn't

Moirai **is** the glue that lets thin, independent agent repos plug into any consuming repo with one command. It defines the shape every agent conforms to, validates the consumer's configuration, and provides the wiring for each LLM host (today: GitHub Actions via `claude-code-action`).

Moirai is **not** an agent host. Each agent lives in its own repo, on its own release cadence. Moirai never owns prompts or scopes — it just orchestrates their installation.

## The three layers

| Layer | What it is | Examples |
|---|---|---|
| **Agent repo** | One agent, one purpose. Required files: `AGENT.md`, `PROMPT.md`, `schema.json`, `defaults.yml`. No workflows, no config values. | [`curator`](https://github.com/derrybirkett/curator), more to come |
| **Moirai (this repo)** | Registry, protocol spec, host adapters, installer. The substrate that lets thin agents work the same way everywhere. | this repo |
| **Consuming repo** | Submodules Moirai. Per agent, has `.github/agents/<name>/config.yml` (validated against the agent's `schema.json`) + a workflow per trigger. | [`monospace.studio`](https://github.com/derrybirkett/monospace.studio), future adopters |

## Quick start

Add Moirai as a submodule and install an agent:

```bash
git submodule add https://github.com/derrybirkett/moirai .moirai
.moirai/bin/install curator
```

`bin/install` reads [`registry.yml`](registry.yml), submodules the agent at its pinned tag, scaffolds `.github/agents/<name>/config.yml` from the agent's `defaults.yml`, and drops a workflow file from the chosen adapter template.

Tune the generated config (it's a one-PR change in your consuming repo, no submodule dance), and you're live.

## What's in this repo

- **[`protocol/SPEC.md`](protocol/SPEC.md)** — the contract every agent and adapter must follow. Required files in an agent repo, config shape rules, environment variable envelope, sink interface, adapter responsibilities.
- **[`registry.yml`](registry.yml)** — the catalog of known agents, their repos, their pinned tags, and the adapter that wires each one.
- **[`adapters/claude-code-action/`](adapters/claude-code-action/)** — GitHub Actions adapter using [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action). Today's only adapter. Future: an OpenAI Assistants adapter, a local CLI runner.
- **[`bin/install`](bin/install)** — the installer. Bash. Reads registry, drops files into the consuming repo.

## Interop pattern — why thin agents compound

Every agent writes to a **typed sink**: `issues` | `inbox` | `summary` | `pr-comment`. Every agent reads other agents' outputs via stable schemas (e.g. Curator's `[curator] <category>:` issue titles, Auditor's `## Audit — … (PR #N)` blocks in `notes/inbox.md`).

This means agents don't have to know about each other to compose. A future meta-agent reads across sinks and proposes cross-cutting action. **Thin agents stay thin; interop comes from the substrate.**

## Status

Walking-skeleton, v0.1.0. The protocol works for one agent (Curator) end-to-end. Step 3 of the rollout plan refactors `monospace.studio` to consume Curator via Moirai; step 4 adds Auditor as the second agent — the moment of truth for whether the protocol accommodates more than one cleanly.

## License

MIT — see [LICENSE](LICENSE).
