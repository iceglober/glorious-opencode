# @glrs-dev/harness-opencode

An opinionated OpenCode agent harness delivered as a single npm package. Two usage modes:

1. **OpenCode plugin** — registers agents, slash commands, tools, MCPs, and skills into your OpenCode session. Interactive, human-in-the-loop.
2. **Pilot CLI** — runs a task DAG fully unattended against a real OpenCode server. Plan a ticket, validate the DAG, execute it, review the results.

Both modes ship in the same package. You can use one or both.

---

## Mode 1: OpenCode Plugin

### What you get

When OpenCode starts, the plugin injects:

- **14 agents** — `orchestrator` (five-phase end-to-end), `plan` (interactive planner), `build` (plan executor), `qa-reviewer`, `qa-thorough`, `plan-reviewer`, `gap-analyzer`, `code-searcher`, `architecture-advisor`, `docs-maintainer`, `lib-reader`, `agents-md-writer`, `pilot-builder`, `pilot-planner`
- **7 slash commands** — `/ship`, `/autopilot`, `/review`, `/init-deep`, `/research`, `/fresh`, `/costs`
- **5 custom tools** — `ast_grep`, `tsc_check`, `eslint_check`, `todo_scan`, `comment_check`
- **5 MCP servers** — `serena` (AST code intel), `memory` (per-repo JSON memory), `git` (structured blame/log). `playwright` and `linear` defined but disabled by default.
- **5 skill bundles** — `review-plan`, `web-design-guidelines`, `vercel-react-best-practices`, `vercel-composition-patterns`, `pilot-planning`
- **4 sub-plugins** — `autopilot` (opt-in completion loop), `notify` (OS notifications), `cost-tracker` (LLM spend tracking), `pilot-plugin` (runtime invariant enforcement)

### Install

```bash
bunx @glrs-dev/harness-opencode install
```

This adds `"@glrs-dev/harness-opencode"` to the `plugin` array in `~/.config/opencode/opencode.json`. Your existing plugins and settings are preserved. A `.bak` backup is written before any mutation.

Or add it manually:

```json
{
  "plugin": ["@glrs-dev/harness-opencode"]
}
```

No global install required. OpenCode's plugin loader resolves the package from its own cache.

### Use

```bash
opencode
```

That's it. The `orchestrator` agent is the default. All agents, commands, tools, MCPs, and skills are available immediately.

### Customize

**Agents, commands, MCPs:** your `opencode.json` overrides win. To swap the orchestrator model:

```json
{
  "agent": {
    "orchestrator": {
      "model": "anthropic/claude-sonnet-4-6"
    }
  }
}
```

**Model tiers:** override all agents in a tier at once via `harness.models`:

```json
{
  "harness": {
    "models": {
      "deep": ["bedrock/claude-opus-4"],
      "mid": ["bedrock/claude-sonnet-4"],
      "fast": ["bedrock/claude-haiku-4"]
    }
  }
}
```

Per-agent overrides in `harness.models` win over tier. Direct `agent.<name>.model` overrides in `opencode.json` win over everything.

**Skills:** read-only by design (they live in `node_modules`). To customize, fork the package.

---

## Mode 2: Pilot CLI

The pilot subsystem runs a `pilot.yaml` task DAG fully unattended. You define tasks with dependencies, touch-globs, and verify commands; the pilot worker executes them in topological order using isolated git worktrees.

### Prerequisites

Everything from Mode 1 (the plugin must be installed so the `pilot-builder` and `pilot-planner` agents are available), plus:

- `git` >= 2.5 (for `git worktree` support)
- `opencode` CLI on PATH

### Install

The plugin install from Mode 1 is sufficient — the pilot CLI ships in the same package. To invoke it:

```bash
# Via bunx (no global install needed)
bunx @glrs-dev/harness-opencode pilot <verb>

# Or install globally for a shorter command
bun add -g @glrs-dev/harness-opencode
glrs-oc pilot <verb>
```

The global install is optional but recommended if you use pilot regularly. It also avoids a known issue where `bunx` can spawn Node.js instead of Bun on some systems.

### Quick start

```bash
# 1. Create a plan interactively (spawns OpenCode TUI with the pilot-planner agent)
glrs-oc pilot plan "Add user auth with OAuth"

# 2. Validate the plan (schema, DAG cycles, glob conflicts)
glrs-oc pilot validate

# 3. Run the plan (single worker, isolated worktrees, fully unattended)
glrs-oc pilot build

# 4. Check progress
glrs-oc pilot status
```

### CLI verbs

| Verb | Description |
|------|-------------|
| `plan [input]` | Spawn the OpenCode TUI with the `pilot-planner` agent. Input can be a Linear ID, GitHub URL, or text description. |
| `validate [path]` | Validate a `pilot.yaml` against schema, DAG, and glob rules. Defaults to newest plan. |
| `build` | Run the pilot worker against a plan. `--plan <path>`, `--dry-run`, `--filter <id>`. |
| `status` | Print the current run's task statuses. `--run <id>`, `--json`. |
| `resume` | Continue a partially-completed run. Skips succeeded tasks. |
| `retry <task-id>` | Reset a single task to pending and optionally re-run. `--run-now`. |
| `logs <task-id>` | Print events and verify outputs for a task. |
| `worktrees list\|prune` | List or prune git worktrees managed by pilot. |
| `cost` | Print per-task and total LLM cost for a run. `--json`. |
| `plan-dir` | Print the resolved plan directory path (creates if missing). |

### State storage

All pilot state lives under `~/.glorious/opencode/<repo>/pilot/`:

```
pilot/
  plans/                  # YAML plans (input artifacts)
  runs/<runId>/
    state.db              # SQLite (runs, tasks, events)
    workers/00.jsonl      # per-worker structured logs
  worktrees/<runId>/00/   # git worktree for task execution
```

The `<repo>` segment is derived from `git rev-parse --git-common-dir`, so worktrees of the same repo share state. Override with `$GLORIOUS_PILOT_DIR`.

> **v0.1 limitation:** single worker only. `--workers >1` clamps to 1. Multi-worker parallel scheduling is deferred to v0.3+.

---

## Shared operations

### Update

```bash
bun update @glrs-dev/harness-opencode
```

Or if you're using floating semver in `opencode.json`, OpenCode's internal `bun install` handles it on startup.

### Health check

```bash
bunx @glrs-dev/harness-opencode doctor
# or
glrs-oc doctor
```

### Pin to a specific version

```bash
bunx @glrs-dev/harness-opencode install --pin
```

Injects `"@glrs-dev/harness-opencode@<current-version>"` into your plugin array.

### Rollback a broken release

```bash
npm deprecate @glrs-dev/harness-opencode@<broken> "<reason>"
```

Then ship a patch. Users on floating semver auto-recover on next `bun update`.

### Uninstall

```bash
bunx @glrs-dev/harness-opencode uninstall
```

Removes the plugin entry from `opencode.json`. To also remove the package: `bun remove @glrs-dev/harness-opencode`.

## Migrating from the old clone+symlink install

If you were using the previous `install.sh`-based harness, see [docs/migration-from-clone-install.md](docs/migration-from-clone-install.md).

## Privacy

The plugin checks `registry.npmjs.org` once per day for newer versions. No analytics, no telemetry, no identifiers beyond what `fetch()` sends. Opt out: `export HARNESS_OPENCODE_UPDATE_CHECK=0`.

## Prerequisites

- [OpenCode](https://opencode.ai)
- `bun` (for plugin installation and CLI)
- `uvx` (for serena + git MCPs — `brew install uv`)
- `node`/`npx` (for memory MCP)
- `git` >= 2.5 (for pilot worktrees)

## Contributing

Pull requests welcome. Read [`AGENTS.md`](./AGENTS.md) for plugin architecture, type-surface escape hatches, and the zero-user-filesystem-writes invariant.

All user-visible PRs require a changeset: `bunx changeset` before opening the PR. See [`CONTRIBUTING.md`](./CONTRIBUTING.md).

## License

MIT
