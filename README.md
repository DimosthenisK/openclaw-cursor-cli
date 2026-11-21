# openclaw-cursor-cli (fixed fork)

An OpenClaw plugin that adds a `cursor-cli` CLI backend and provider, so OpenClaw can route model calls through the local `cursor-agent` binary (using your Cursor subscription).

> **This is a fork of [`jeehou/openclaw-cursor-cli`](https://github.com/jeehou/openclaw-cursor-cli).**
> Upstream 0.0.6 does not work on current OpenClaw (2026.10.x / 2026.11.x): requests either
> fail with `Cannot use this model: <family>`, `Invalid URL: must be http or https`,
> `Unknown CLI backend`, or `CLI stream-json output ended without a result event`.
> See [Upstream failures and what this fork fixes](#upstream-failures-and-what-this-fork-fixes)
> for the exact cause of each and the one-line-to-few-line repair.

**Fixes carried in this fork (on top of upstream 0.0.6)**

| Symptom on upstream | Root cause | Fix here |
| --- | --- | --- |
| `Cannot use this model: claude-opus-4-8` (and every non-`auto` model) | `model-rules.json` never existed, so catalog family ids were forwarded verbatim | Startup rebuild of the rules cache (+ a warning when a model is unmapped) |
| `Invalid URL: must be http or https` / silent fallback to another model | `buildCliBackend()` omitted `modelProvider`, so OpenClaw dropped the provider→CLI-runtime binding | `modelProvider: "cursor-cli"` |
| `CLI stream-json output ended without a result event` on turn 2+ | `resumeArgs` replaced `args`, dropping `-p`/`--output-format stream-json` | `resumeArgs: [...args, "--resume", "{sessionId}"]` |
| `Unknown CLI backend: cursor-cli` after a restart | `activation.onStartup: false` + `api: "openai-completions"` in provider config | `activation.onStartup: true` + `onProviders` |
| Control UI: "0 of N models available" even when logged in | No synthetic-auth seam and no non-secret auth marker | `prepareSyntheticAuth` + `syntheticAuthRefs` + `nonSecretAuthMarkers` |
| `--trust` rejected by newer cursor-agent | Flag set was from an older CLI | Trust/force flags stay opt-in via `allowTools` |
**Pattern**: structurally identical to OpenClaw's built-in `claude-cli` backend — a thin wrapper that spawns the Cursor IDE's official CLI in headless `-p --output-format stream-json` mode and pipes the result back into OpenClaw.

**Highlights**

- All Cursor models (`auto`, `composer-2`, GPT-5.x, Claude 4.5/4.6/4.7, Gemini 3.x, Grok, Kimi, …) usable via `cursor-cli/<base>`.
- OpenClaw's unified `--thinking off|low|medium|high|xhigh|max` flag is translated at runtime to the corresponding Cursor model id (`-thinking-high`, `-extra-high`, etc.). Catalog stays small (~13 user-facing base ids); per-effort variants are derived.
- Model catalog and thinking-level rules are **fetched live** from `cursor-agent models` and cached — no hardcoded lists to drift when Cursor adds models.
- Single `/cursor-models refresh` slash command (in supported channels) or `bash scripts/refresh-models.sh` (shell) to re-sync after Cursor updates.

## Security & permissions — read first

This plugin acts as a thin CLI bridge to cursor-agent and inherits all of cursor-agent's capabilities. Choose a permission profile that matches your trust level before enabling it for real workflows:

- **Using this purely as a model provider?** Set `mode:"ask"` + `allowTools:false` so cursor-agent runs in read-only Q&A and refuses any tool action. See the [Recommended safety profiles](#recommended-safety-profiles) table below.
- **Working in an untrusted workspace?** Add `sandbox:"enabled"` to confine tool actions to cursor-agent's sandbox, and/or use `mode:"plan"` to prevent file edits.
- **Want the same behavior as OpenClaw's built-in `claude-cli` backend?** Leave config empty — but only do so in workspaces you trust, since cursor-agent can then use its own write/shell tools.
- Model calls run under your **local `cursor-agent` login** and consume your **Cursor subscription quota**. Make sure `cursor-agent status` shows the Cursor account you actually intend to bill.
- Session resume is enabled so multi-turn conversations work. If you switch between sensitive and non-sensitive tasks in the same channel, start a fresh OpenClaw conversation/session for the sensitive one.

## Requirements

- OpenClaw `>= 2026.5.7`
- `cursor-agent` installed on PATH and logged in:

  ```bash
  cursor-agent login         # opens browser OAuth, once
  cursor-agent status        # should show ✓ Logged in as ...
  cursor-agent -p "say hi"   # smoke test — must return text
  ```

## Install

### From this fork (recommended)

```bash
git clone https://github.com/DimosthenisK/openclaw-cursor-cli.git
openclaw plugins install ./openclaw-cursor-cli
```

### Replacing an existing upstream install

```bash
openclaw plugins install ./openclaw-cursor-cli --force
openclaw gateway restart
```

The plugin id stays `cursor-cli`, and `plugins.entries."cursor-cli"`, the
`models.providers.cursor-cli` catalog, and the `agents.defaults.models` allowlist
carry over unchanged.

### Post-install (both routes)

```bash
# 1. Verify it loaded cleanly. Your existing default model is untouched.
openclaw plugins doctor             # "No plugin issues detected."
openclaw plugins inspect cursor-cli # cli-backend + text-inference both registered
openclaw models                     # Default unchanged

# 2. Fetch the live Cursor catalog, family rules, and auto-allowlist all
#    cursor-cli/<family> entries in agents.defaults.models.
#    Optional on this fork: the rules cache is rebuilt at startup when it is
#    missing or older than 7 days. Run it explicitly to also write the
#    `models.providers.cursor-cli` catalog into openclaw.json.
bash ~/.openclaw/extensions/cursor-cli/scripts/refresh-models.sh
```

After step 2:

- `openclaw models list --provider cursor-cli` shows every family with the correct context window (1024k / 256k / 200k).
- `/model` (slash command in chat channels) lists every `cursor-cli/<family>` as a selectable option.
- Re-running step 2 is safe: it only **adds** missing entries to `agents.defaults.models` and never removes or overwrites entries you (or other plugins) already put there.

## Usage

### Per-call (recommended — zero default change)

```bash
openclaw agent --local --agent main \
  --model cursor-cli/claude-opus-4-7 \
  --thinking high \
  --message "design a worker pool" \
  --json
```

The `--thinking` flag is automatically translated:

| --thinking | claude-opus-4-7 → cursor model id |
|---|---|
| off | `claude-opus-4-7-medium` |
| low | `claude-opus-4-7-thinking-low` |
| medium | `claude-opus-4-7-thinking-medium` |
| high | `claude-opus-4-7-thinking-high` |
| xhigh | `claude-opus-4-7-thinking-xhigh` |
| max | `claude-opus-4-7-thinking-max` |

(Mapping for every family is in `~/.openclaw/extensions/cursor-cli/model-rules.json` after refresh. View with `bash scripts/refresh-models.sh --dry-run` if curious.)

### Make Cursor your default

```bash
openclaw models set cursor-cli/claude-opus-4-7
# revert any time:
openclaw models set deepseek/ali-deepseek-v4-pro
```

## Refreshing when Cursor adds new models

Whenever Cursor ships new model ids, run **one** of:

**Shell**:
```bash
bash ~/.openclaw/extensions/cursor-cli/scripts/refresh-models.sh
```

The script auto-locates your `openclaw` npm package so the refresh helper can reuse OpenClaw's plugin-sdk subprocess helper (no `child_process` import on our side). If auto-detection fails — e.g. unusual nvm / yarn layout — override with:
```bash
OPENCLAW_PACKAGE_ROOT=/path/to/lib/node_modules/openclaw \
  bash ~/.openclaw/extensions/cursor-cli/scripts/refresh-models.sh
```

**Chat slash command** (in channels that support plugin slash commands — telegram / discord / feishu / mattermost):
```
/cursor-models refresh
```

This runs the same logic in-process inside the plugin runtime — no shell wrapper, no env-var tricks.

Both routes:
1. Run `cursor-agent models` and parse all 100+ Cursor model ids.
2. Group ids into families (auto, gpt-5.5, claude-opus-4-7, …) and infer per-family suffix patterns automatically.
3. Build a `thinking-level → real cursor id` mapping for each family.
4. Persist the catalog to `models.providers.cursor-cli` (your OpenClaw config).
5. Persist the family-rules cache to `~/.openclaw/extensions/cursor-cli/model-rules.json` (read by the plugin's `resolveExecutionArgs` hook at runtime).
6. Merge each `cursor-cli/<family>` into `agents.defaults.models` so the `/model` slash command and OpenClaw agent allowlist see them. Only **adds** missing entries — existing entries (yours or other plugins') are left untouched.

Idempotent — re-running just updates the same files. OpenClaw backs up `openclaw.json` to `.bak` before every config write.

### Other `/cursor-models` sub-commands (slash only)

| Sub-command | Description |
|---|---|
| `/cursor-models` or `/cursor-models status` | Show cache file path, age, family count, cursor id count. |
| `/cursor-models refresh` | Re-fetch and rebuild as described above. |
| `/cursor-models list` | Print every cached family with its `off` and `high` mapping. |
| `/cursor-models help` | Show usage. |

## Plugin config (optional)

All fields are optional. Defaults preserve full agent mode (backward compatible with `0.0.1`). In `~/.openclaw/openclaw.json`:

```json5
{
  plugins: {
    entries: {
      "cursor-cli": {
        enabled: true,
        config: {
          // Path or basename of the cursor-agent binary (default: cursor-agent on PATH).
          // Only an absolute path or a simple basename like "cursor-agent" is accepted;
          // shell metacharacters (`;`, `|`, `&`, `` ` ``, `$`, `<`, `>`, …) are rejected.
          command: "~/.local/bin/cursor-agent",

          // Execution mode. Pick the lowest privilege your workflow needs.
          //   "agent" (default) — full agent, can plan + edit + run shell tools
          //   "plan"            — read-only/planning, proposes diffs but never edits
          //   "ask"             — pure Q&A, no tool use at all
          mode: "agent",

          // Controls whether cursor-agent may run its write/shell tools without
          // interactive approval. Leave it at the default to keep the standard
          // OpenClaw cli-backend headless behavior (matches `claude-cli`). Set
          // to false in untrusted workspaces — every tool action will then
          // require explicit confirmation. Pairs naturally with mode:"plan"/"ask".
          allowTools: true,

          // Optional override for cursor-agent's sandbox setting.
          //   "enabled"  — confine tool actions to a sandbox.
          //   "disabled" — opt out (cursor-agent's own default behavior).
          //   omitted    — leave it to cursor-agent's config.
          // sandbox: "enabled",

          // Advanced: extra args appended verbatim. Most users do not need this.
          extraArgs: []
        }
      }
    }
  }
}
```

### Recommended safety profiles

Pick the lowest privilege your workflow actually needs.

| Profile | Config | Effect |
| --- | --- | --- |
| **Read-only Q&A** *(recommended for pure provider use)* | `{ mode: "ask", allowTools: false }` | Pure Q&A; cursor-agent refuses any tool action. |
| **Plan-only** | `{ mode: "plan", allowTools: false }` | Read-only/planning; proposes diffs but never writes. |
| **Sandbox-enforced** | `{ sandbox: "enabled" }` | Headless backend confined to cursor-agent's sandbox. |
| **Full agent** *(legacy default — matches `claude-cli`)* | `{}` or omit | Headless cli-backend; tool actions auto-approved inside the workspace. Use only in trusted workspaces. |

## Upgrading from earlier versions

If you previously installed `0.0.1` – `0.0.4`, the upgrade is safe and mostly hands-off. Default plugin behavior is unchanged (omitting `mode`/`allowTools`/`sandbox` keeps the original `0.0.1` semantics) and nothing outside `cursor-cli/*` in your config is touched.

Run **two** commands after `openclaw plugins install clawhub:@jeehou/openclaw-cursor-cli --force`:

```bash
openclaw doctor --fix                                                # 1) replay config migrations
bash ~/.openclaw/extensions/cursor-cli/scripts/refresh-models.sh      # 2) re-sync the live catalog
```

| Step | What it fixes | Required for |
| --- | --- | --- |
| `openclaw doctor --fix` | Auto-merges every `cursor-cli/<id>` already in `models.providers.cursor-cli` into `agents.defaults.models`. Strictly additive — never removes or overrides existing entries. | Users on `0.0.1`–`0.0.4` whose `/model` picker was missing `cursor-cli/*` entries. |
| `refresh-models.sh` | Rewrites the provider catalog from live `cursor-agent models` output (correct context windows, new families released since you last refreshed) and re-applies the allowlist merge. | Anyone seeing wrong `contextWindow` (e.g. `claude-4.5-opus` reading 195k instead of 200k) or missing newly-launched models. |

`openclaw security audit` will also report a `cursor-cli.cache.*` finding (`missing` / `legacy` / `stale`) when the model-rules cache predates `0.0.5` or is older than 30 days, so you can run the refresh on demand instead of on a schedule.

## Uninstall / Rollback

```bash
# Light: disable, keep installed
openclaw plugins disable cursor-cli

# Full uninstall
openclaw plugins uninstall cursor-cli

# Also drop the provider config, the rules cache, and the allowlist entries
# the refresh script added:
openclaw config unset 'models.providers.cursor-cli'
rm -f ~/.openclaw/extensions/cursor-cli/model-rules.json
# Strip cursor-cli/* keys from agents.defaults.models (other entries preserved):
PRUNED=$(openclaw config get 'agents.defaults.models' \
  | python3 -c 'import json,sys; d=json.load(sys.stdin); print(json.dumps({k:v for k,v in d.items() if not k.startswith("cursor-cli/")}))')
openclaw config set 'agents.defaults.models' "$PRUNED"
```

Other providers (`deepseek`, `modelhub`, `openai-codex`, …) are never touched by any of this.

## Architecture

```
openclaw agent --local --model cursor-cli/claude-opus-4-7 --thinking high --message "…"
    ↓
OpenClaw resolves model ref → provider=cursor-cli
    ↓
Routes to CLI backend "cursor-cli" (registered by this plugin)
    ↓
plugin.resolveExecutionArgs reads ~/.openclaw/extensions/cursor-cli/model-rules.json,
   looks up rules.families["claude-opus-4-7"]["high"] → "claude-opus-4-7-thinking-high"
    ↓
Invokes cursor-agent in headless print mode with the stream-json output format
(plus optional flags derived from plugin config: execution mode, sandbox setting,
non-interactive tool approval, session resume) and the resolved Cursor model id.
    ↓
Parses NDJSON stream via OpenClaw's claude-stream-json dialect:
   {type:"system",subtype:"init",session_id}     → session bind
   {type:"thinking",subtype:"delta",text}         → (dropped — Claude dialect doesn't know)
   {type:"assistant",message:{content:[{text}]}}  → assistant text
   {type:"result",subtype:"success",result,...}   → finalize
    ↓
Returns to OpenClaw as a normal assistant turn (session_id persisted for resume)
```

No HTTP, no OAuth, no protobuf reverse engineering. All auth, rate limiting, and billing remain with the official `cursor-agent` CLI.

## File layout

```
openclaw-cursor-cli/
├── package.json
├── openclaw.plugin.json          # manifest (id, providers, cliBackends, commandAliases)
├── index.js                      # definePluginEntry: registerCliBackend + registerProvider + registerCommand
├── src/
│   └── refresh-models.mjs        # parse cursor-agent → infer family rules → write user config + cache
├── scripts/
│   └── refresh-models.sh         # shell entry point (used by install + manual refresh)
└── README.md
```

## Limitations

## Upstream failures and what this fork fixes

Diagnosed against OpenClaw 2026.11.6 (06afb10) with `cursor-agent`
2026.11.19-f005e9d on Linux, plugin 0.0.6 from ClawHub.

### 1. `Cannot use this model: <family>` — the rules cache was never built

The catalog published to `models.providers.cursor-cli` lists *family* ids
(`claude-opus-4-8`, `gpt-5.6-sol`, `cursor-grok-4.6`, …). Real `cursor-agent`
accepts only *concrete* ids (`claude-opus-4-8-high`, `gpt-5.6-sol-medium`,
`cursor-grok-4.6-high`, …). The translation table lives in
`~/.openclaw/extensions/cursor-cli/model-rules.json`, written only by
`scripts/refresh-models.sh` / `/cursor-models refresh`.

When that file is absent, `loadRules()` falls back to a two-entry stub
(`auto`, `composer-2`). Every other family id is passed through unchanged and
cursor-agent answers:

```text
Cannot use this model: cursor-grok-4.6. Available models: auto, ...
```

`auto` keeps working, which makes this look like a per-model availability
problem instead of a missing cache.

**Fix here:** `ensureRulesCacheFresh()` rebuilds the cache at startup when it is
missing, malformed, or older than 7 days (by invoking `cursor-agent models`
directly, ~1.4 s, without touching `openclaw.json`). An unmapped model now also
logs the exact remediation instead of failing silently.

### 2. `Invalid URL: must be http or https` — missing `modelProvider`

`buildCliBackend()` returned `{ id, config, liveTest, nativeToolMode,
resolveExecutionArgs }` with no `modelProvider`. OpenClaw creates the
provider→CLI-runtime binding only from that field, and drops the registration
without a warning when it is absent. `cursor-cli/*` calls then reach the generic
HTTP transport with `baseUrl: "cli://cursor-agent"` and fail on URL validation,
or silently fall back to another model.

**Fix here:** `modelProvider: "cursor-cli"` on the backend, plus
`subscriptionAuthDispatch: true` so embedded runs on the local `cursor-agent`
login execute through the CLI instead of an HTTP passthrough.

### 3. `CLI stream-json output ended without a result event` — broken resume args

OpenClaw replaces `args` with `resumeArgs` on every resumed turn. Upstream set
`resumeArgs: ["--resume", "{sessionId}"]`, which drops `-p`,
`--output-format stream-json`, and the trust flags. cursor-agent then runs
without print mode, emits no `result` event, and OpenClaw reports a truncated
stream.

**Fix here:** `resumeArgs: [...args, "--resume", "{sessionId}"]`, the same
construction the bundled `claude-cli` and `google-gemini-cli` backends use.

### 4. `Unknown CLI backend: cursor-cli` after a restart — activation filter

With `activation.onStartup: false`, and `models.providers.cursor-cli.api` set to
a core built-in API id (`openai-completions`), the Gateway start-up planner
concluded that no plugin needed to load. The manifest still advertised
`cliBackends: ["cursor-cli"]`, so dispatch went to a backend that had never been
registered.

**Fix here:** `activation.onStartup: true` + `onProviders: ["cursor-cli"]`.

### 5. Control UI "0 of N models available" — no synthetic auth

Cursor authenticates through its own local login, so there is no OpenClaw
credential. The Control UI and catalog then classify every `cursor-cli` model as
unavailable and route model picks into provider setup.

**Fix here:** `prepareSyntheticAuth` probes `cursor-agent status --format json`
(never reading token material) and reports a non-secret marker declared in the
manifest as both `syntheticAuthRefs` and `nonSecretAuthMarkers`.

### Not a bug: there is no system-prompt transport

`cursor-agent` exposes `--system-prompt`, but Cursor rejects it for normal
accounts:

```text
RetriableError: [invalid_argument] Custom system prompt is not enabled for this account.
```

There is no `--append-system-prompt` and no system-prompt file flag. The
supported channel is the agent's workspace instructions file (`AGENTS.md` in the
run directory), which cursor-agent loads natively. `systemPromptWhen` is
therefore `never` in this fork; configuring a flag that does not exist only
produces CLI usage errors.

- **Streaming**: uses `--output-format stream-json --stream-partial-output` and the `claude-stream-json` jsonl dialect (Cursor's NDJSON is event-compatible with Claude Code's). Multi-turn session resume via `--resume <id>` works.
- **`thinking` events from Cursor are silently dropped** by the `claude-stream-json` parser (Claude's dialect doesn't know that event type). Final `assistant` and `result` lines are parsed correctly — only intermediate reasoning text is invisible.
- **Token usage counters** are not surfaced into OpenClaw's `agentMeta.usage` under stream-json. Switch `output: "jsonl"` → `"json"` in `index.js` if you need accurate accounting.
- **By default the plugin runs cursor-agent in OpenClaw's standard headless cli-backend profile** (same shape as the built-in `claude-cli` backend), which lets cursor-agent use its own write/shell tools inside the current workspace. For untrusted workspaces, switch to the safer profiles described under *Recommended safety profiles* above (e.g. `mode:"ask"` + `allowTools:false`, or `sandbox:"enabled"`).
- **Per-`-fast` variants** (e.g. `gpt-5.5-high-fast`) are not exposed in the catalog yet. Add a separate `--fast` flag axis if needed.
- **External plugins can't register top-level `openclaw <cmd>` CLI subcommands** in OpenClaw 2026.5.7 — that's why refreshing from the shell goes through `scripts/refresh-models.sh` rather than e.g. `openclaw cursor-models refresh`. The `/cursor-models` slash command is the in-chat equivalent.
