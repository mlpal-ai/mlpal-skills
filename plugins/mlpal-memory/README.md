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
pip install --index-url https://pypi.mlpal.ai/simple/ --extra-index-url https://pypi.org/simple/ mlpal-cli
mlpal login
mlpal memory connect claude-code
```

Codex users: `mlpal memory connect codex` configures `~/.codex/config.toml`; copy the skill folder
into `~/.codex/skills/`.

## What it writes, and where

Everything lands in your personal store on MLPal, named with the tool that wrote it. You see and edit
the same memory on [mlpal.ai](https://mlpal.ai). A write that carries a credential is refused; writes
are capped per request and per day.
