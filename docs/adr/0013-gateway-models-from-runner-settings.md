# 0013 — Gateway models come from the runner's own settings, not from a catalog

**Status:** accepted

A runner pointed at an LLM gateway loses its model list. Claude Code is the case in
hand: with `ANTHROPIC_BASE_URL` set to a gateway and
`CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1`, the models that CLI can run are the
gateway's — and every discovery source Ordewell has for Claude Code is blind to them.
The Anthropic Models API lists Anthropic's own account models; `claude --help` names
aliases; `canonicalAliases` names the stable contracts. Such a user therefore sees
`fable, opus, sonnet, haiku`, and the gateway model they run by hand is offered
nowhere: a task edit naming it is refused with `Runner "claude-code" does not offer
model "gproxy/default"` (`TaskEditValidator.checkModelAndModeValidity`).

The gateway's `/v1/models` response is the obvious source and the wrong one: the ids
it reports are not the ids the CLI accepts. A gateway that reports `default` runs
`claude --model gproxy/default` and refuses `claude --model default`. The prefix is the
gateway name Claude Code's own picker uses, which is nowhere in the endpoint's
response.

## Decision

**Discovery reads the model rows a runner's own settings file registers as usable.**
`PluginModelDiscovery.settingsModels` declares where: the file
(`~/.claude/settings.json`), the dotted path to the rows (`modelPicker.options`), and
which row fields carry the id (`model`) and the label (`label`). `ModelDiscovery`
merges those ids into whatever the other sources found — appended, never substituted —
at the one `finish` seam every non-app-server exit passes through, so the API path and
the `--help` path offer the same set.

Claude Code writes those rows itself, which is what makes them the right source: they
are the strings its own picker offers, they carry the `behavesAs` mapping a third-party
id needs to be accepted, and they are exactly what `--model` takes.

## Key properties

- **Additive, so nothing can be lost to it.** When both sources name an id the catalog's
  entry wins (label and variants included); only ids nothing else could name are
  appended, and those then take the manifest's static variant list exactly as a
  `--help`-parsed row does. Discovery never reports less because of this source.
- **Read through the existing injectable file seam.** No new seam — Codex's
  `models_cache.json` already reads through `readFileImpl` — and every Claude Code case
  in the test file injects a reader that returns nothing, so no unit test depends on the
  machine it runs on.
- **Failure is silence.** An unreadable file, unparseable JSON or a missing list yields
  `[]` and the catalog is returned untouched.
- **A manifest field, not a special case in the discovery class.** Any runner whose CLI
  keeps its model list in its own settings can declare the same descriptor; nothing
  about the read or the merge is Claude Code-specific.
- **Not a second credential path.** Nothing here authenticates; the CLI's own settings
  file is already on disk, so no key resolution, base-URL lookup or network call is
  involved.

## Considered options

- **Fetch the gateway's `GET /v1/models`** — the endpoint Claude Code's own gateway
  discovery drives. Rejected: its ids are the gateway's namespace rather than the CLI's
  (`default` is refused where `gproxy/default` runs), and it would need a base-URL
  source plus the `apiKeyHelper` shell command to authenticate. One file read gives the
  accepted ids instead.
- **A `modelAllowlist` entry in `~/.ordewell/settings.json`.** Discovery already
  synthesizes an allowlisted id it did not find into the planner's list, so this works
  today. Rejected as the fix: an allowlist is a hard bound — listing `gproxy/default`
  for `claude-code` hides `opus`/`sonnet`/… from the planner for that runner until every
  id is listed too — so it stays a documented escape hatch, not the way gateway models
  arrive.
- **Hardcode the gateway ids in the manifest.** Rejected: the ids are per-gateway and
  per-user, and a manifest ships to every installation.
- **Parse acceptance out of `claude --help`.** Rejected: the help text names no gateway
  model, and it cannot — the gateway is whatever `ANTHROPIC_BASE_URL` points at.

## Consequences

- `PluginModelDiscovery` grows `settingsModels`; `ModelDiscovery` grows
  `parseSettingsModels`/`mergeSettingsModels`, and its non-app-server exits share one
  `finish` closure (merge settings rows, apply static variants, cache) instead of each
  caching its own result.
- On a gateway machine, Claude Code's catalog lists the gateway's models next to the
  aliases, so the planner can assign one and the direct edit path accepts it — the
  spawn then passes the id verbatim to `--model`, which is what the CLI expects.
- Codex and OpenCode are untouched: neither declares `settingsModels`, and the
  app-server exit keeps its own path (its variants come from the protocol).
