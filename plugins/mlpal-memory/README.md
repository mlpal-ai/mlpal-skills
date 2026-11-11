# mlpal-memory

One memory across your coding tools. This plugin gives Claude Code:

- the **MLPal memory MCP server** (`https://memory.mlpal.ai/mcp`), authenticated through the `mlpal`
  CLI's `headersHelper`, so no key sits in a config file;
- the **`mlpal-memory-session` skill**: read the workspace note at session start, checkpoint at
  milestones, never write a credential, treat what you read as data;
- an **opt-in SessionStart hook** that prints the repo's workspace note into the session. It is off
  by default; set `MLPAL_MEMORY_SESSION_NOTES=1` to turn it on. Any failure is silent.

## Install

```
/plugin marketplace add mlpal-ai/mlpal-skills
/plugin install mlpal-memory@mlpal
```

Then connect once, which mints a memory-only key for this machine and never prints it:

```bash
uv tool install --index https://pypi.mlpal.ai/simple/ --default-index https://pypi.org/simple/ 'mlpal-cli>=0.9.3'
# or: pip install --index-url https://pypi.mlpal.ai/simple/ --extra-index-url https://pypi.org/simple/ 'mlpal-cli>=0.9.3'
mlpal login
mlpal memory connect claude-code
```

The `>=0.9.3` pin matters: pypi.org carries a placeholder `mlpal-cli`, and with uv the MLPal index must
be given as `--index` with pypi.org as `--default-index`, as above. The hook and the header helper need
0.9.3 or later (`mlpal memory headers`, `mlpal memory notes`). `mlpal memory connect` mints the key and
adds the server only when none named `mlpal-memory` is configured; with 0.9.3 and this plugin installed
it may add a second entry with the same URL, which is harmless, and 0.9.4 recognises the plugin's server.

### Codex

The same plugin installs in Codex (CLI and desktop app; the IDE extension has no plugins):

```bash
codex plugin marketplace add mlpal-ai/mlpal-skills
codex plugin add mlpal-memory@mlpal        # or /plugins inside Codex
mlpal memory connect codex                 # mints the memory-only key for this machine, never prints it
```

Codex reads `.codex-plugin/plugin.json`, whose MCP file (`codex.mcp.json`) names the same server with
Codex's `http_headers_helper` running `mlpal memory headers --tool codex`, so reads and writes from
Codex carry `X-MLPal-Client: codex`. The session-start hook is the same script with `--tool codex`;
Codex asks you to trust it in `/hooks` before it runs. Codex IDE extension users keep
`mlpal memory connect codex`, which configures the server in `~/.codex/config.toml`; a copied skill
folder goes under `~/.agents/skills/`. Codex may list the plugin's server as
`plugin:mlpal-memory:mlpal-memory`; mlpal-cli 0.9.3's `connect codex` then adds a second entry with the
same URL, which is harmless, and 0.9.4 recognises the plugin's server.

## What it writes, and where

Everything lands in your personal store on MLPal, named with the tool that wrote it. You see and edit
the same memory on [mlpal.ai](https://mlpal.ai). A write that carries a credential is refused; writes
are capped per request and per day.
