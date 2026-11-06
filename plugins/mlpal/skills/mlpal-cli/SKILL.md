---
name: mlpal-cli
description: Use when running the `mlpal` command-line tool. Covers installing it, signing in (browser, headless/SSH, profiles), the wallet and usage, gateway API keys (create, list, revoke), models, the cloud workspace and HOPs. Use it when the user wants to manage their MLPal account from a terminal or a script.
---

# The mlpal CLI

`mlpal` is the MLPal platform CLI. Everything below is for version **0.8.4 or later**; check with `mlpal --version`.

## Install

```bash
pip install --index-url https://pypi.mlpal.ai/simple/ --extra-index-url https://pypi.org/simple/ mlpal-cli
# or: uv tool install --index-url https://pypi.mlpal.ai/simple/ --extra-index-url https://pypi.org/simple/ mlpal-cli
mlpal --version
```

## Sign in

```bash
mlpal login                    # opens the browser; works with Google sign-in
mlpal login --no-browser       # SSH or headless: prints a link and a code to approve on any device
mlpal login -p work            # a named profile (-p is always --profile)
mlpal login --password -u you@example.com   # email + password instead of the browser
mlpal whoami                   # who you are signed in as
mlpal profiles                 # every stored profile and whether it can authenticate
mlpal logout                   # revoke the session and delete what is stored
```

- Use a profile on any command with `--profile NAME` (or `-p NAME`), or set `MLPAL_PROFILE=NAME`.
- Only approve a login code that came from a terminal **you** started. Anyone holding the code could act as you.

**Who a command runs as** (the first match wins):
1. `MLPAL_CREDENTIAL_HELPER`, inside an MLPal workspace.
2. `MLPAL_ACCESS_TOKEN`, a delegated session such as the dashboard assistant.
3. `--profile NAME` or `MLPAL_PROFILE`.
4. `MLPAL_API_KEY`. Only `assistants models` and `assistants usage` accept it; every other command refuses rather than fall back to your stored login.
5. Your stored default login.

## Wallet and usage

```bash
mlpal wallet                   # balance, card on file, and what to do next
mlpal usage                    # compute units (CU) used, with the wallet
mlpal usage cost               # CU and USD, per source
mlpal usage by-model           # CU per model
mlpal usage by-product         # CU and USD per product
mlpal usage trend              # month by month
mlpal usage rate               # current CU price
```

Managed MLPal uses a prepaid wallet measured in compute units (CU). A new account needs a verified card before gateway keys work. `mlpal wallet` tells you when that's the case and links to billing.

## AI Gateway: keys and models

```bash
# Create a key and write it straight into .env. The secret is never printed.
mlpal assistants keys create --name my-app --env-file .env --env-var MLPAL_API_KEY
# or into a file (mode 0600):
mlpal assistants keys create --name my-app --secret-file ./mlpal.key

mlpal assistants keys list                 # your keys and their status (active / expired / revoked / inactive)
mlpal assistants keys usage                # per-key usage
mlpal assistants keys revoke <id>          # revoke a key
mlpal assistants models list               # every model you can route to
mlpal assistants models get <model-tag>    # one model, with pricing
mlpal assistants models aliases            # meta aliases (e.g. `mlpal`) and what they route to
mlpal assistants usage                     # gateway requests, tokens and CU this month (--days N for per day)
mlpal assistants health                    # gateway health
```

Secret rules:
- `keys create` refuses to print a new key into a pipe or a non-terminal (exit 8). Use `--env-file` or `--secret-file`, or `--reveal` to get a one-time reveal handle that works for 5 minutes.
- Never paste a key into chat, a commit, a log or a command line. Read it from the environment.
- To call the gateway with the key, see the `mlpal-gateway` skill.

## Workspace (your cloud dev box)

```bash
mlpal workspace status         # status, size, sleep timer, HOPs
mlpal workspace sizes          # size classes and prices
mlpal workspace open           # link that wakes it and opens it in the browser
mlpal workspace start --wait   # wake it and wait until it's ready
mlpal workspace stop           # sync files, then stop billing
mlpal workspace size <size>    # change size (a running workspace restarts)
mlpal workspace sleep <mins>   # idle sleep timer
mlpal workspace always-on on   # never sleep (needs a verified card)
mlpal workspace usage          # this month: hours per size, CU, USD
```

## HOPs (harness operating profiles)

```bash
mlpal hops search <text>               # search the platform catalog
mlpal hops show <name>[@version]       # a catalog HOP and its versions
mlpal hops get-started coding          # make a catalog HOP usable in your scope
mlpal hops list                        # catalog HOPs and your registrations
mlpal hops chat <slug> "hi"            # one message to a published HOP
mlpal hops keys --help                 # HOP keys (mlpal_hk_...)
```

## Scripting

- Add `-o json` for machine output. stdout carries exactly one JSON document; hints go to stderr.
- Exit codes:

| Code | Meaning |
|---|---|
| 0 | ok |
| 1 | failed |
| 2 | bad usage |
| 3 | not signed in, or the session expired (run `mlpal login`) |
| 4 | permission or payment needed |
| 5 | not found |
| 7 | invalid input |
| 8 | secret output refused |
| 69 | service unavailable |
| 75 | temporary, so retry |
| 77 | delegated session ended |
| 130 | cancelled |

- Every group has `--help`. When unsure, run `mlpal <group> --help` rather than guessing flags.

## Other groups

These follow the same pattern; check `--help` on each:
- `mcp`: MCP servers
- `skills` and `plugins`
- `storage`
- `secrets`
- `db`: managed Postgres
- `schedules`: run a HOP on a timetable
- `org`
- `projects`
- `metrics`

`static-agents`, `da` and `sites` are listed under "Closed for now".
