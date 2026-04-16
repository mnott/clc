# clc

Claude Code has an env-var problem. Every project needs different model settings,
version pins, and token limits — but there's no clean way to manage that across
projects without manually editing JSON files every time you switch contexts. clc fixes this.

Install clc and you get a three-layer config system that merges a shared user baseline,
project-level overrides, and runtime CLI flags into a single resolved launch command.
Switch projects and your settings follow. Pin a version for one repo without touching
another. Set haiku as the subagent model globally and override it locally for the one
project that needs opus. Everything composable, nothing magic.

## Quick Start

Tell Claude Code:

> Clone https://github.com/mnott/clc and set it up for me

That's it. Claude clones the repo, reads this README, and runs `./clc install` for you.

### Or do it manually

```bash
git clone https://github.com/mnott/clc.git dev.clc
cd dev.clc
./clc install
```

`clc install` runs `uv sync` and symlinks the script into `~/.local/bin/` (falling back
to `/usr/local/bin/` with sudo only if `~/.local/bin` isn't on PATH).

Requires `uv` on PATH (`brew install uv` on macOS). Python itself is managed by uv —
no separate install needed.

To remove: `clc uninstall` (or tell Claude "uninstall clc" from inside the repo).

### Verify

```bash
clc config show        # see effective values across all layers
clc --dry-run          # preview the full launch command without running it
clc                    # launch Claude Code with your current config
```

---

## Why Three Layers

Managing Claude Code parameters by editing `~/.claude/settings.json` by hand works
until you have five projects with different needs. Then it becomes a source of
breakage — you forget which setting you changed for which project, or a global
change clobbers a local override you didn't remember you'd made.

clc makes the layering explicit:

| Layer  | File                                  | Purpose                          |
|--------|---------------------------------------|----------------------------------|
| user   | `~/.claude/settings.json` (env block) | Shared baseline across projects  |
| global | `<clc dir>/mysettings.json`           | clc-provided opinionated defaults |
| local  | `./.claude/mysettings.json`           | Per-project overrides (highest)  |
| CLI    | flags passed to `clc` / `clc run`     | Runtime-only, never persisted    |

Later layers override earlier ones. CLI flags beat everything and are never written
to disk. The local layer is the most specific — it wins because it knows the most
about the current project.

`clc config show` shows you every managed parameter, where the effective value comes
from, and what the other layers have set. No guessing.

---

## Quickstart Examples

Pin a specific Claude Code version for this session only:

```bash
clc --version 2.1.96
```

Preview the full launch command without running it:

```bash
clc run -v 2.1.96 --dry-run
```

Persist a version pin for this project only:

```bash
clc config set CLAUDE_CODE_VERSION 2.1.96 --local
```

Use sonnet with the 1M-token context window:

```bash
clc --model sonnet[1m]
```

Use haiku for subagents to save costs, globally:

```bash
clc config set CLAUDE_CODE_SUBAGENT_MODEL haiku --user
```

Override it to opus for a specific high-stakes project:

```bash
clc config set CLAUDE_CODE_SUBAGENT_MODEL opus --local
```

Disable the 1M context window for a project that doesn't need it:

```bash
clc config set CLAUDE_CODE_DISABLE_1M_CONTEXT 1 --local
```

---

## Managed Parameters

clc tracks 23 parameters, defined in `params.json` next to the script.
Use `clc config show` to see effective values across all layers, or
`clc config show -v` for descriptions and examples. Edit `params.json` to add
or retire parameters — no code changes required.

| Parameter | What it controls | Examples |
|-----------|-----------------|---------|
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Autocompact trigger as a percentage of the context window. | 50, 80, 92 |
| `CLAUDE_CODE_CACHE_TTL_MS` | Prompt-cache TTL in milliseconds. 86400000 = 24h. | 300000 (5m), 3600000 (1h), 86400000 (24h) |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | Force-disable the 1M-token context window regardless of model (1 = disable, 0 = enable). | 0, 1 |
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` | Disable adaptive thinking (1 = disable, 0 = enable). | 0, 1 |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | Disable automatic memory loading (1 = disable, 0 = enable). | 0, 1 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Disable telemetry, error reporting, feature-flag polling, and other background network calls. May suppress some server-gated experimental features. | 0, 1 |
| `CLAUDE_CODE_EFFORT_LEVEL` | Reasoning depth per turn. 'max' forces full reasoning every turn. Overrides the settings.json effortLevel field (which caps at 'high'). Note: the 'ultrathink' prompt trigger maps to 'high', so with max set it is a downgrade. | low, medium, high, max |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | Enable experimental multi-agent coordination tools (TeamCreate, TaskCreate/Update/List, SendMessage). WARNING: ~7x token cost when agent teams are actively used. | 0, 1 |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Maximum output tokens per assistant turn. | 4096, 16384, 32768, 64000 |
| `CLAUDE_CODE_MIN_TPS` | Minimum acceptable tokens-per-second before the client escalates or warns. | 100, 200, 300 |
| `CLAUDE_CODE_MODEL` | Model to launch with. Plain 'opus'/'sonnet' DISABLES the 1M-token context; add [1m] to enable it. | opus, opus[1m], sonnet, sonnet[1m], haiku, (empty = harness default) |
| `CLAUDE_CODE_MODEL_PERF` | Model performance tier hint. | predictable, fast, default |
| `CLAUDE_CODE_NO_FLICKER` | Disable terminal redraw flicker on slow renders (1 = disable flicker, 0 = default). | 0, 1 |
| `CLAUDE_CODE_NO_NERF` | Disable model nerfs / capability dampening (1 = disable nerfs, 0 = default). | 0, 1 |
| `CLAUDE_CODE_RATE_LIMIT_ENABLED` | Toggle client-side rate limiting (1 = enable, 0 = disable). | 0, 1 |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Default model used by subagents spawned via the Task tool. | haiku, sonnet, opus |
| `CLAUDE_CODE_VERSION` | Pin a specific Claude Code release (empty = use local binary at ~/.local/bin/claude). | 2.1.96, 2.1.110, (empty) |
| `DANGEROUSLY_SKIP_PERMISSIONS` | Pass --dangerously-skip-permissions to claude (1=yes, 0=no). | 0, 1 |
| `DEFAULT_PROMPT` | Prompt sent as the first message if none passed on CLI. Empty string = send no prompt. | go, continue, (empty) |
| `ENABLE_TOOL_SEARCH` | Enable deferred tool loading via ToolSearch (1 = on, 0 = off). | 0, 1 |
| `IS_SANDBOX` | Advertise that Claude is running in a sandboxed environment. | 0, 1 |
| `MAX_THINKING_TOKENS` | Upper bound on internal reasoning ('scratch pad') tokens per turn. 31999 is the documented maximum. With CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1 this becomes the fixed budget rather than a ceiling. | 10000, 20000, 31999 |
| `USER_TYPE` | User-type marker consumed by internal tooling. | ant, external |

---

## Common Recipes

Show every managed parameter with its effective source:

```bash
clc config show
clc config show -v    # include descriptions and examples
```

Set autocompact to trigger at 80% context:

```bash
clc config set CLAUDE_AUTOCOMPACT_PCT_OVERRIDE 80 --user
```

Reset all managed keys in user settings.json back to clc's global defaults:

```bash
clc config reset
```

Initialize a per-project local config from global defaults:

```bash
clc setup
```

Regenerate this README:

```bash
clc doc > README.md
```

---

## CLI Reference

```
clc [OPTIONS]               Launch Claude Code with effective config.
clc run [OPTIONS] [PROMPT]  Explicit launch subcommand; same options as above.

Options shared by clc and clc run:
  -v, --version VERSION     Pin a Claude Code release via npx.
  -m, --model MODEL         Model to launch (overrides config, not persisted).
  -e, --effort LEVEL        Effort level: low | medium | high | max.
  -t, --max-tokens TOKENS   Max output tokens (shorthand: 4k, 16k, 32k, 64k).
  -s, --subagent SUBAGENT   Subagent model: haiku | sonnet | opus.
  -n, --dry-run             Print the command that would run, then exit.
      --skip-perms / --no-skip-perms  Toggle --dangerously-skip-permissions.

clc config show [-v] [-x]   Show all managed parameters and their values.
clc config set KEY VALUE [--local | --user | --global]
                            Set a parameter in the chosen layer.
clc config unset KEY [--local | --user | --global]
                            Remove a parameter from the chosen layer.
clc config reset [-y]       Reset user settings.json to clc global defaults.

clc setup [--force]         Create ./.claude/mysettings.json from global defaults.
clc install [-f]            Run uv sync and symlink clc into ~/.local/bin/ (or /usr/local/bin/).
clc uninstall [-f]          Remove symlink(s) created by clc install.
clc doc                     Print this README as markdown (clc doc > README.md).
```
