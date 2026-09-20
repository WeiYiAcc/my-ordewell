# Agent memory in Ordewell — where facts belong, per usage mode

Design note for this fork. It exists because Ordewell's own planner recommended keeping
sops/dotfiles facts in `ORDEWELL.md`, which cannot work: that file is capped at 8000
characters and the cut is silent.

## What Ordewell actually injects (verified against `packages/core`)

| Channel | Where | Cap | Who sees it |
|---|---|---|---|
| `ORDEWELL.md` | workspace root | **8000 chars**, `slice()`, no warning | planner system prompt, always |
| Directory tree | `listDir('.', 3)` | 300 lines (`tree`) / 200 entries (flat) | planner system prompt, always |
| Runner agent config | manifest `contextFile` | 3000 chars — **dead code** | nobody: `ContextCollector.setRegistry()` has no caller, so `agentConfig` is always null |
| `AGENTS.md` / `CLAUDE.md` | workspace root | none (read tool: 1 MB per call) | planner **reads it on demand** — `PlanPrompts.ts` opens the research phase with "Start by reading README.md and any agent config files (AGENTS.md, CLAUDE.md)" |
| Skills | `~/.ordewell/skills/`, `<ws>/.ordewell/skills/` (local shadows global) | none | planner **user message** only, on `/name`; never in the system prompt, never in a runner prompt |
| Runner-native config | `CLAUDE.md` (claude-code), `AGENTS.md` (codex, opencode) | none | the runner CLI reads it itself, at spawn |

Consequences:

- `ORDEWELL.md` is a 8000-char **preamble**, not a store. Overflow is dropped silently,
  in UTF-16 units, so a cut can split a surrogate pair.
- `AGENTS.md` is the real always-reachable channel: the planner is *told* to read it, and
  the runners read it natively.
- Skills are the only **cross-project** channel — there is no global context file in
  `~/.ordewell/`, only settings, `.env`, `plugins/`, `skills/`.

## Per usage mode

| Mode | How context reaches the planner | `/skill-name` in a message | What to use for durable memory |
|---|---|---|---|
| CLI, one-shot (`plan --no-chat`) | `ORDEWELL.md` + tree; `AGENTS.md` read only because the prompt tells it to | **no** — this path goes through `generatePlan`, which never calls `resolveSkillInvocation` | `ORDEWELL.md` index + `AGENTS.md` + `docs/` |
| CLI, conversational (default) | same, via `startPlanning` | **yes** | the above, plus `/machine-memory` for cross-repo facts |
| TUI | same core, private daemon (stopped on exit); skills registered as slash commands | **yes** | same as conversational CLI |
| API / programmatic host | host-controlled: inject `SessionDeps.skillsService`, `aiService` or `planner` | depends on the method the host calls (`startPlanning`/`continueConversation` expand, `generatePlan` does not) | inject memory at runtime — the only place secrets may live |
| VS Code | the same core, bundled (`tsup noExternal`); workspace root = **first** folder only | **yes** (webview has its own token UI) | workspace `ORDEWELL.md` + `.ordewell/skills/`; the 13 settings are runtime knobs, not context |

Long output that must survive: write it to a file in the workspace and point at it. The
planner reads workspace files freely (`read_file`, `grep`); only paths **outside** the
workspace need an approval.

## Secrets

`ORDEWELL.md`, `AGENTS.md`, `CLAUDE.md` and every skill body are sent verbatim to whichever
planner backend is active. `ORDEWELL.md` is also not gitignored. So: record *how* to use
sops (commands, key paths, repo names), never the key material. Decrypt in the host and
inject through `skillsService` / `aiService` when a secret must actually reach a model.

## Installing the `machine-memory` skill

The source of truth is `skills/machine-memory/SKILL.md` in this repo:

```bash
ln -sfn "$PWD/skills/machine-memory" ~/.ordewell/skills/machine-memory   # global
# or, per project: cp -r skills/machine-memory <project>/.ordewell/skills/
```

Invoke it as `/machine-memory` in the TUI, the conversational CLI, and the VS Code chat.
It is a no-op for `plan --no-chat`, which does not expand skill tokens.

## Known gap that looks like a bug: a finished runner that never exits

A runner that finishes its work but stays alive at its own prompt keeps the task in
`in_progress` forever — no marker, no verdict, and every dependent task stays blocked.
Observed on a multi-task plan: task 1 passed, a later subtask's Claude Code session sat
idle at its own prompt, and that task never resolved.

Why: silence is deliberately not a verdict.

- `VerdictEngine.ts` — `IDLE_TIMEOUT_MS = 60_000`, documented as "advisory, UI-only".
  `touchIdle` only stamps `idleSince`; nothing in that file fails a task on silence.
- `TaskOrchestrator.ts` — `onIdleChange` is wired to `emit('onTaskChanged')` only:
  "idleSince is advisory UI state, not a store mutation". `getIdleSince` is a getter.
- So a task resolves in exactly two ways: its completion marker, or its exit code. An
  interactive runner that never exits after a missed marker hangs instead of failing.

Where a fix belongs (open, and a product decision): the idle transition is already an
event (`VerdictEngine.onIdleChange` / `IdleListener`), so a configurable grace period
could fail the task through the same missing-marker path the rest of the engine uses.
It is not done because a legitimately long-thinking task must not be killed — the window
is a judgement call. Part of the fix may also belong in the runner shape: an interactive
runner that has finished its work should arguably exit.

Practical consequence: when a task sits `in_progress` with a live but silent runner pane,
read the pane before concluding that the model, the runner or the gateway is broken.
