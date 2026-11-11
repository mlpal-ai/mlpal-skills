# MLPal Skills

These are official [Agent Skills](https://agentskills.io) for building on [MLPal](https://mlpal.ai). Each skill teaches your coding agent how to use one part of the platform correctly, with examples we've tested against production. That covers keys, endpoints, the SDK, the CLI, errors and billing.

| Skill | What it covers |
|---|---|
| [`mlpal-gateway`](plugins/mlpal/skills/mlpal-gateway/SKILL.md) | The MLPal AI Gateway: API keys, the chat API, the Anthropic-compatible Messages API, OpenAI-compatible embeddings, the Python SDK, model routing (`model="mlpal"`), streaming, usage and costs in compute units, and errors |
| [`mlpal-cli`](plugins/mlpal/skills/mlpal-cli/SKILL.md) | The `mlpal` CLI: install, sign in (browser, headless, profiles), the wallet and usage, gateway keys and models, the cloud workspace, HOPs, and scripting with JSON output and exit codes |
| [`mlpal-sites`](plugins/mlpal/skills/mlpal-sites/SKILL.md) | MLPal Sites (invite-only): static sites and small web apps at `<slug>.mlpal.app`, deploys and review holds, rollbacks, custom domains, usage and billing, and scripting with exit codes |

This is one plugin, and we keep adding skills to it. To get new ones, run `/plugin marketplace update mlpal`.

A second plugin, [`mlpal-memory`](plugins/mlpal-memory/README.md), gives your coding tools one shared memory on MLPal: the memory MCP server, the session protocol skill and an opt-in session-start hook. Install it with `/plugin install mlpal-memory@mlpal`.

## Install

### Claude Code

```
/plugin marketplace add mlpal-ai/mlpal-skills
/plugin install mlpal@mlpal
```

Ask Claude something like *"call gpt through the MLPal gateway"* and the skill loads by itself. To pick up new skills later, run `/plugin marketplace update mlpal`.

You can also install without the marketplace: copy a skill folder into `~/.claude/skills/` (for every project) or `.claude/skills/` (for one project).

```bash
git clone https://github.com/mlpal-ai/mlpal-skills
cp -r mlpal-skills/plugins/mlpal/skills/mlpal-gateway ~/.claude/skills/
```

### Other agents and harnesses

Every skill is a plain folder with a `SKILL.md` file (YAML front matter plus Markdown), in the open Agent Skills format. Any harness that reads skills can use it:

- **OpenAI Codex:** `codex plugin marketplace add mlpal-ai/mlpal-skills`, then `codex plugin add <plugin>@mlpal` (or `/plugins`); a copied skill folder goes under `~/.agents/skills/`.
- **yodex and the MLPal harness:** copy it into `.yodex/skills/` in your project, or install the plugin.
- **Cursor, Windsurf, Cline, Continue and others:** add the `SKILL.md` content to the agent's rules or context file, or point the agent at the file.
- **Your own agent:** load `SKILL.md` into the system prompt when a request matches its `description`.

## What you need

- An MLPal account at [mlpal.ai](https://mlpal.ai), with a verified card. Usage is metered in compute units (1 CU = US$10) from a prepaid wallet, and model tokens are priced at the provider's list price.
- An API key, either from **Dashboard → Assistants → API keys** or from the CLI:

  ```bash
  pip install --index-url https://pypi.mlpal.ai/simple/ --extra-index-url https://pypi.org/simple/ mlpal-cli
  mlpal login                      # opens the browser; on a headless machine it prints a link and a code
  mlpal assistants keys create --name my-app --env-file .env --env-var MLPAL_API_KEY
  ```

  The key is written into `.env` and never printed. Keep keys in environment variables, and never commit them.

## Contributing

Each skill lives in `plugins/mlpal/skills/<name>/SKILL.md`. Every code example has to run against production before it's merged. Open an issue or a pull request.

## License

Apache-2.0
