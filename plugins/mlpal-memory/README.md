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

Codex users: `mlpal memory connect codex` configures `~/.codex/config.toml`; copy the skill folder
into `~/.codex/skills/`.

## What it writes, and where

Everything lands in your personal store on MLPal, named with the tool that wrote it. You see and edit
the same memory on [mlpal.ai](https://mlpal.ai). A write that carries a credential is refused; writes
are capped per request and per day.
