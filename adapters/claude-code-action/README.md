# adapter: claude-code-action

GitHub Actions adapter for Moirai. Wraps [`anthropics/claude-code-action@v1`](https://github.com/anthropics/claude-code-action) — the agent's prompt is loaded from `$AGENT_REPO_DIR/PROMPT.md`, the envelope (§3 of the protocol spec) is injected, and the LLM runs with the tools declared in the agent's `AGENT.md`.

## What this adapter handles

- **Prompt loading** — reads `$AGENT_REPO_DIR/PROMPT.md`, appends the runtime envelope as a "Runtime context" section, and passes it to the action.
- **Envelope injection** — sets `AGENT_NAME`, `AGENT_CONFIG_PATH`, `AGENT_REPO_DIR`, `TODAY` as env vars before the LLM runs.
- **Auth** — passes `secrets.ANTHROPIC_API_KEY` via the action's `with: anthropic_api_key` input (not via `env:`, which the action ignores), and `secrets.GITHUB_TOKEN` via `with: github_token`.
- **Tool allow-list** — pulled from the agent's `AGENT.md` (typically `Bash, Read` plus sink-specific tools).
- **Kill switch** — checks for `.<agent-name>-pause` at repo root and exits early if present.

## Required setup in the consuming repo

| Requirement | How |
|---|---|
| `ANTHROPIC_API_KEY` repo secret | `gh secret set ANTHROPIC_API_KEY` |
| Workflow permission `id-token: write` | Already in the generated workflow file. |
| Workflow permission `contents: read` (or `write` for `inbox` sink) | Already in the generated workflow file based on declared sink. |
| Workflow permission `issues: write` (for `issues` sink) | Already in the generated workflow file based on declared sink. |

## Template

[`workflow.yml.template`](workflow.yml.template) is the file the installer drops into `.github/workflows/<agent>.yml`. Placeholders (`{{AGENT_NAME}}`, `{{TRIGGER_BLOCK}}`, `{{PERMISSIONS}}`, `{{ALLOWED_TOOLS}}`) are replaced by `bin/install`.

## What this adapter is not

- Not a Claude Code MCP server.
- Not an agent itself. It's the wiring around the agent.
- Not portable to non-GitHub-Actions hosts. A separate adapter would be needed (e.g. `adapters/local-cli`, `adapters/openai-assistants`).
