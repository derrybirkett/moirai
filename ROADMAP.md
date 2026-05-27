# Moirai Roadmap

Living document of where Moirai is now, where it's heading, and the architectural reasoning behind the next layers.

Pairs with [`CHANGELOG.md`](CHANGELOG.md) (what shipped) and [`protocol/SPEC.md`](protocol/SPEC.md) (the current contract).

---

## Where we are — `v0.3.0`

The walking-skeleton is complete and validated end-to-end:

- Three sink+trigger combos supported: `schedule + issues` (Curator), `pull_request + inbox` (Auditor per-PR, opt-in), `schedule + inbox` (Auditor nightshift, default).
- Two agents in the registry: [Curator](https://github.com/derrybirkett/curator), [Auditor](https://github.com/derrybirkett/auditor).
- Two consuming repos: [`monospace.studio`](https://github.com/derrybirkett/monospace.studio), [`deltado`](https://github.com/derrybirkett/deltado).
- One host adapter: `claude-code-action` on GitHub Actions.

The protocol's atomic unit is the **agent**. One agent ↔ one workflow file ↔ one trigger ↔ one sink. Adopting an agent in a new repo is one `bin/install <name>` command.

## Where it's heading — three layers above the current atomic unit

The current architecture optimises for *shipping individual agents*. The next architectural concerns are *composing them into coordinated work* and *scaling them across hosts*. Three new primitives, introduced in order of cost-benefit:

### Layer 1 — **Jobs-to-be-done framing for agents**

Today an agent's `AGENT.md` describes *what it does* (persona, output rules, what it isn't). It does not describe *what job it's hired to do*. Adding an explicit `job:` field to the AGENT.md frontmatter buys three things:

1. **Discovery** — "what agent does the X job?" becomes a registry query, not a docs search.
2. **Competitive substitution** — different agents can be candidates for the same job (e.g. two different "tidy the repo" agents with different scopes), evaluated against outcomes.
3. **Outcome-based evaluation** — agents are measured by whether their job got done, not by how many turns they used.

**Risk:** JTBD too abstract. Discipline: keep jobs at the recurring-pattern granularity, not the per-task one.

**When:** add to the protocol the next time we extract an agent (Advisor likely). Backfill Curator + Auditor's AGENT.md at the same time. Cheap.

### Layer 2 — **Stories (declarative orchestration)**

Today the unit is the agent. The natural next primitive is the **story** — a named, recurring event-response pattern that coordinates multiple agents.

A story bundles:
- One or more triggers (schedule, pull_request, webhook)
- An ordered (or parallel) list of agents to run
- Per-story config: budget envelope, termination criteria, retry policy
- A name and purpose ("Nightshift", "Pre-merge coherence check", "Incident triage")

Today all of that lives implicitly across N separate workflow files. A story file would consolidate them.

**Concrete shape:**

```yaml
# moirai/stories/nightshift.yml
name: nightshift
purpose: "End-of-day repo hygiene — tidy debt + surface coherence drift"
trigger:
  schedule: "0 3 * * *"
agents:
  - curator      # serial: curator first
  - auditor      # then auditor
budget:
  max_cost_usd_per_run: 5
  max_wall_clock_minutes: 30
on_failure: continue   # one agent failing doesn't block others
```

The Moirai installer generates ONE workflow per story (not one per agent). Consuming repos pick which stories to install. Migrating today's agents into stories is mechanical: each existing per-agent workflow becomes a one-agent story.

**Why this first:** stories are a config-shape change, not an architecture change. Same GitHub Actions substrate. Cheap to try, easy to revert if the layering's wrong. Validates the pattern before committing to the bus.

**When:** Moirai `v0.4.0`. Required by direction C (Conductor) — which is "a story that reads Curator + Auditor's sinks and decides cross-cutting action."

### Layer 3 — **Skills (capability primitives)**

Today each agent's prompt contains its own scan-codebase logic, its own write-inbox logic, its own file-issue logic. These are conceptually the same primitives across agents. Factoring them into named, versioned **skills** that agents *invoke* would give:

- Smaller agent prompts (focused on judgment, not execution mechanics)
- Shared improvements (fix `scan-codebase` once, every agent benefits)
- Cross-agent consistency (Curator and Auditor's diff-set logic stay aligned)
- Easier evaluation (test a skill in isolation)

**Concrete shape:**

```yaml
# moirai/skills/scan-codebase.yml
name: scan-codebase
purpose: "Walk scan_paths, return all files matching the agent's filter"
inputs: { scan_paths, exclude_paths, file_patterns }
outputs: { files: [{ path, last_modified, ... }] }
```

Agents declare which skills they use in `AGENT.md` (`skills: [scan-codebase, write-inbox]`); the installer ensures versions are compatible.

**Risk:** changes the agent contract. Higher migration cost. Premature if agents stay this small.

**When:** Moirai `v0.5.0`. After the third or fourth agent exists and the duplication is concrete pain.

### Layer 4 — **Event bus (dispatcher + subscriptions)**

Today triggers are pinned per-workflow at install time. Future: an event dispatcher where stories subscribe to events, and events come from anywhere — cron, webhooks, upstream agent emissions, manual dispatch.

**Concrete shape:**

```
EVENT          cron tick · webhook · state-change · manual dispatch · upstream agent emission
   ↓
DISPATCHER     subscription table: event → stories
   ↓
STORY          declarative orchestration: agents + order + budget + termination + retry
   ↓
AGENTS         thin LLM workers, each with one job
   ↓
SKILLS         capability primitives reusable across agents
   ↓
TOOLS          raw implementation (gh CLI, git, fs, LLM provider, MCP servers)
```

The dispatcher is the missing primitive that lets agents *emit* events (not just consume them). With it, you get:

- **Reactive chains.** Curator files an issue → emits `issue.filed` → Conductor's story responds → proposes a bundled cleanup PR.
- **Cross-repo coordination.** An event on one repo can fire stories in another.
- **Pluggable event sources.** Today's only event sources are cron + GitHub PR events. Adding "deploy completed" or "incident severity 2 declared" becomes a new event type, not a new architecture.

**Concrete substrate options:**
- **Cheap**: GitHub Issues as event log; dispatcher polls or reacts via webhooks. Free, lossy, latency-bound.
- **Real**: a small hosted dispatcher service (Vercel function, Cloudflare Worker, hosted Postgres queue). Costs money, gives ordering + retries + persistence.

**Risk:** breaks the "Claude-first portability" promise if the dispatcher becomes hosted. Mitigation: keep the dispatcher as a *protocol* (event JSON shape, subscription contract) with multiple implementations possible.

**When:** Moirai `v1.0.0` or later. Earliest if a clear need emerges from stories+Conductor work (e.g. "Conductor needs to react to Curator's output without polling"). Otherwise defer until there's pressure.

---

## Recommended path: incremental, story-first

| Version | Adds | Driver |
|---|---|---|
| `v0.4` | Stories (one new YAML file per story; installer generates one workflow per story) | Required by Conductor (direction C) |
| `v0.5` | Skills (factor agent prompts into named capabilities) | Required when 3rd/4th agent exposes real duplication |
| `v0.6+` / `v1.0` | Event bus + dispatcher | Required when stories can't express reactive chains |

The principle: **don't promote a layer until the layer below has surfaced concrete friction that justifies it.** This is the same walking-skeleton discipline that got us from `v0.1.0` to `v0.3.0` — extract one, prove the loop, fix what the friction taught us.

## Decision deferred — leapfrog vs. incremental

The alternative is **leapfrog**: design `v1.0` around events from the start, deprecate the per-agent workflow shape. Bigger jump, cleaner architecture, but no longer a walking-skeleton — you're betting on a design before exercising it.

The case for leapfrog: today's per-agent shape was already deprecated by stories (the v0.4 work below); doing it twice is waste. The case against: each layer revealed concrete bugs in the abstractions above it. Skipping that learning likely surfaces in production as a v1.1 or v1.2 emergency.

**Current preference: incremental.** Revisit if a strong external pressure (multi-repo coordination need, non-GitHub-Actions host) makes leapfrog the right move.

## Open questions

1. **Where do `pickup` / `handover` skills live?** Currently `~/.claude/`, personal-level. Once Layer 3 (skills) ships, do they migrate into Moirai? Or stay as a separate "user-facing skills" layer outside the agent-orchestrator concern?
2. **Cross-LLM portability — when?** Layer 4's dispatcher is where this question gets answered. Today aspirational; remains so until concrete demand.
3. **Story-level cost budgets — enforce at which layer?** Per-story `max_cost_usd_per_run` is in the v0.4 sketch above, but enforcement requires the dispatcher (you need to *track* spend across runs, not just declare a cap). Practical implementation likely waits for Layer 4.
4. **Event vocabulary — who curates it?** A canonical event names list (PR opened, deploy completed, incident declared) needs to live somewhere stable. Probably in `protocol/SPEC.md` once Layer 4 ships.
5. **Backward compatibility across these layers.** Today consumers pin Moirai by tag and inherit changes via submodule bump. As the protocol grows, deprecation paths need to be more deliberate than "just bump major." A `protocol/migrations/` directory or similar.

---

*Last updated: 2026-05-27. Architecture conversation captured during the v0.3.0 wrap-up session.*
