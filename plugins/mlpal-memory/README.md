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
uv tool install 'mlpal-cli>=0.9.4'     # or: pip install 'mlpal-cli>=0.9.4'
mlpal login
mlpal memory connect claude-code        # one memory key per machine, in the keyring, never printed
```

The plugin's server entry names the tool statically (`X-MLPal-Client: claude-code`) and takes the
`Authorization` header from `mlpal memory headers`, which reads the machine's device key; mlpal-cli 0.9.4
or later is needed for that bare form (0.9.3 understood only `--tool`). With 0.9.4, `connect` sees the
plugin's server and adds nothing of its own.

### Codex

The same plugin installs in Codex (CLI and desktop app; the IDE extension has no plugins). **Codex
0.148.0 or later**: earlier versions accept the header helper setting and silently send no key.

```bash
codex plugin marketplace add mlpal-ai/mlpal-skills
codex plugin add mlpal-memory@mlpal        # or /plugins inside Codex
mlpal memory connect codex                 # the same device key; adds nothing when the plugin is enabled
```

Codex reads `.codex-plugin/plugin.json`, whose MCP file (`codex.mcp.json`) names the same server with
`http_headers = {"X-MLPal-Client": "codex"}` and Codex's `http_headers_helper` running
`mlpal memory headers`. The session-start hook is the same script with `--tool codex`; Codex asks you to
trust it in `/hooks` before it runs. `codex exec` (non-interactive) refuses MCP writes unless run with
`--approve-for-me`; reads need nothing. On Codex older than 0.148.0, and in the IDE extension,
`mlpal memory connect codex` configures the server in `~/.codex/config.toml` with a key file that the
shell must have loaded; a copied skill folder goes under `~/.agents/skills/`.

## What it writes, and where

Writes land in your personal store, the tenant `user:<your id>`. A learning or a piece of state answers
`scope: "org", scope_id: "user:<id>"`: that is the personal store's tenant-wide scope, not an
organization. Preferences land at `scope: "user"`. Each row carries the tool that wrote it.


Everything lands in your personal store on MLPal, named with the tool that wrote it. You see and edit
the same memory on [mlpal.ai](https://mlpal.ai). A write that carries a credential is refused; writes
are capped per request and per day.
