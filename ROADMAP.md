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

## Next agent — **Refiner**

Before the next protocol layer, the next *agent extraction*. Refiner curates the issue queue that Curator and Auditor file into: it reads existing `[curator] …` and `[auditor] …` issues and acts on the queue itself rather than the codebase.

**Job:** keep the triage queue honest. Close issues whose underlying code no longer exists (work got done incidentally), merge duplicates, re-label stale ones, demote findings that the same agent has re-filed N times without action.

**Shape:**
- Trigger: `schedule` (nightly, after Curator + Auditor)
- Sink: `issues` (closes and comments — same sink, write-back rather than write-new)
- Scope: read-only against the codebase; write-only against the issue queue

**Why this is the right next agent (not a code-writing fixer, not Conductor):**

1. **Validates "agent reads another agent's output and acts" on the safest possible surface.** The queue is structured (stable title prefixes, labels) and the worst-case failure mode is a wrongly-closed issue, not bad code. Compare a per-category fixer (e.g. a "dead exports remover") where the failure mode is a bad PR that wastes review time.
2. **Generates the data the Conductor decision needs.** Conductor (the v0.4–v1.0 direction in Layer 2/Layer 4 below) is "a story that reads sinks and decides cross-cutting action." That design is currently speculative because we don't know what the queue *looks like* after the noise is filtered out. Refiner produces that signal — once it's running, the question "does this queue have bundle-able structure?" becomes answerable from data, not architecture taste.
3. **Tests whether the `issues` sink needs richer semantics** (close, comment, relabel) before we commit to it as the de-facto agent-output sink. Cheaper to discover gaps now than during a Conductor build.

**Rollout — both consuming repos in parallel:**
- Extract Refiner to its own repo following the protocol (`AGENT.md`, `PROMPT.md`, `schema.json`, `defaults.yml`).
- Pin `v0.1.0` in `registry.yml`.
- Install into both `monospace.studio` and `deltado` on the same day. Two consuming repos with different queue compositions is the real test of whether the agent generalises; one repo isn't enough signal.
- Run for ~2 weeks before drawing conclusions about queue shape and the Conductor question.

**What we'll know after Refiner:**
- Whether the protocol needs a new `pr` sink (for a future code-writing fixer) or whether reusing `issues` for write-back is enough.
- Whether per-category fixers are worth building, or whether Refiner-curated queues are tractable enough for direct human action.
- Whether Conductor is solving a real problem or a speculative one.

**Risk:** Refiner over-closes — silently archives a finding that mattered. Mitigation: every close gets a comment with the reasoning before closing, and a `refiner-closed` label so accidental closes are searchable and reversible.

## Open question — drop the `inbox` sink entirely?

Working hypothesis (not yet decided): the `inbox` sink is a v0.2-era artifact that should be collapsed away.

**Evidence against keeping it:**
- v0.3.0 already deprecated the `schedule + inbox` combo for Auditor in response to a concrete bug (nightly inbox commits caused a merge conflict that blocked an unrelated PR — see [CHANGELOG.md](CHANGELOG.md)). The same failure mode applies to any future schedule+inbox agent.
- Refiner (above) curates *issues*. Inbox blocks have stable headers but no real lifecycle (no close, no label, no assignee, no dedup beyond header replacement). Two output substrates means two curation strategies — and only one has a curator.
- The fork-safety carve-out in [SPEC §4](protocol/SPEC.md) exists solely because inbox commit-back can't push to fork PR head refs. Drop the sink, drop the carve-out, simpler spec.

**The one real use case (`pull_request + inbox`)** — per-PR feedback that survives review noise — is better served by the reserved `pr-comment` sink: lives on the PR where conversation already happens, no commit-back machinery, sticky-comment dedup is a solved pattern, no fork-safety problem.

**Proposed collapse:**

| Use case | Trigger | Sink |
|---|---|---|
| Nightly finders | `schedule` | `issues` |
| Per-PR durable finds | `pull_request` | `issues` |
| Per-PR feedback (typical) | `pull_request` | `pr-comment` |
| Dry-run / smoke | any | `summary` |

3 sinks instead of 4. Migration cost is Auditor-side (its `pull_request + inbox` mode swaps to `pr-comment`), not a Moirai protocol break.

**When to decide:** during the Refiner rollout. If Refiner's first two weeks confirm issues are the right substrate for agent output, formalise the inbox drop in Moirai v0.5 alongside the `pr-comment` sink becoming first-class.

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

## Atropos — branch cleanup capability

Atropos currently manages stale issues and PRs. Branch cleanup is a natural extension of its "cut what has gone cold" mandate — but should be reactive, not autonomous, to avoid deleting in-flight work.

**Proposed flow:**

1. **Lachesis marks the branch.** After Lachesis observes a merged PR (or a branch whose PR has closed/squash-merged), it adds an `atropos:delete-branch` label to the closed PR.
2. **Atropos deletes on next run.** Atropos queries closed PRs carrying `atropos:delete-branch`, checks the associated remote branch still exists, and deletes it via `gh api repos/{owner}/{repo}/git/refs/heads/{branch}`.
3. **Safety checks before delete:**
   - Branch must point to a commit already in `main` (i.e. reachable — confirms merge actually landed).
   - Branch must not match a `protect_patterns` list in config (e.g. `beta`, `release/*`).
   - Optionally: skip if branch was pushed within the last N hours (grace window for force-push recovery).

**Config additions (`config.yml`):**

```yaml
categories:
  stale_branches: true          # new — off by default

branch_cleanup:
  protect_patterns:
    - beta
    - release/*
  grace_hours: 2                # skip branches pushed within last N hours
  max_deletes_per_run: 20
```

**Why Lachesis marks, not Atropos deciding alone:**

Atropos deciding autonomously which branches are "done" requires it to understand PR state, merge strategy (squash vs. merge vs. rebase), and whether the commit landed on main. Lachesis already reads all of that. Separation keeps Atropos as executor and Lachesis as observer — consistent with their current trust model (`observe` for both; `execute` for Atropos only).

**When:** after Refiner ships and the issue-sink pattern is validated. Branch cleanup is a lower-risk extension (worst case: a branch is deleted that needed restoring — `git push origin <sha>:refs/heads/<name>` recovers it) than any code-writing capability.

**Protocol implications:** this is the first reactive chain in production — Lachesis labels → Atropos acts. Validates the pattern ahead of Stories (Layer 2) without needing the full dispatcher machinery.

---

## Open questions

1. **Where do `pickup` / `handover` skills live?** Currently `~/.claude/`, personal-level. Once Layer 3 (skills) ships, do they migrate into Moirai? Or stay as a separate "user-facing skills" layer outside the agent-orchestrator concern?
2. **Cross-LLM portability — when?** Layer 4's dispatcher is where this question gets answered. Today aspirational; remains so until concrete demand.
3. **Story-level cost budgets — enforce at which layer?** Per-story `max_cost_usd_per_run` is in the v0.4 sketch above, but enforcement requires the dispatcher (you need to *track* spend across runs, not just declare a cap). Practical implementation likely waits for Layer 4.
4. **Event vocabulary — who curates it?** A canonical event names list (PR opened, deploy completed, incident declared) needs to live somewhere stable. Probably in `protocol/SPEC.md` once Layer 4 ships.
5. **Backward compatibility across these layers.** Today consumers pin Moirai by tag and inherit changes via submodule bump. As the protocol grows, deprecation paths need to be more deliberate than "just bump major." A `protocol/migrations/` directory or similar.

---

*Last updated: 2026-06-23. Added Atropos branch-cleanup capability (Lachesis marks → Atropos deletes); added Refiner as next agent extraction.*
