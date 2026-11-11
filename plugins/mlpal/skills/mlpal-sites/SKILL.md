---
name: mlpal-sites
description: Use when publishing a website or a small web app on MLPal Sites with the `mlpal sites` CLI. Covers invite-only access, the card and wallet requirement, creating a static or service site, deploying a built site or a FastAPI/Node app, review holds, rollbacks, custom domains, usage and billing, deleting, and scripting with exit codes.
---

# MLPal Sites

MLPal Sites hosts a site at `https://<slug>.mlpal.app`. A site is either:
- **static**: a built site (HTML, CSS, JS, images…), served from the CDN;
- **service**: a small server app (FastAPI, Node…), started on demand behind the same CDN.

Everything below is for `mlpal` **0.9.3 or later** (`mlpal --version`). See the `mlpal-cli` skill to install it and sign in.

## Before you start

- **Access is invite-only for now:**
  ```bash
  mlpal sites access     # says whether Sites is open for this account
  mlpal sites waitlist   # ask for an invite; safe to repeat, your first request keeps its place
  ```
  Without an invite, `create` is refused (exit 4) and `list` says so.
- **A verified card and a wallet in good standing are required.** This applies to every site, static or service, and to creating, deploying, rolling back and adding domains. Otherwise the CLI exits 4 with `sites.card_required` or `sites.payment_required`. Add a card and funds in the dashboard wallet first.

## Create a site

```bash
mlpal sites create my-site                    # --kind auto: looks at the current directory
mlpal sites create my-api --dir ./api         # auto-detects from ./api
mlpal sites create my-docs --kind static --spa  # --spa serves /index.html for extensionless paths
```

`--kind auto` picks **service** when the project has a server, and prints why:
- `app.py` or `main.py` imports FastAPI, Flask, Django, Starlette or aiohttp;
- `package.json` has a `start` script.

Otherwise it's **static**.

A project with only a `Procfile` is also detected as a service, but **a Procfile isn't run**. Give it an `app.py`/`main.py` or an npm `start` script, or create the site with `--kind static`.

**The slug:**
- **Shape:** 3–40 characters of `a-z`, `0-9` and `-`, starting and ending with a letter or digit.
- **Refused:** brand names, and common reserved names such as `www`, `app`, `docs`, `blog`, `test`, `dev`, `status` and `news`.
- **Not reissued:** once a site has served, its slug is never given out again, even after it's deleted. A slug whose site never went live frees up 30 days after deletion.

An account has at most 3 sites, at most 10 site creations and 10 deletions a day, and 30 deploys an hour.

## Deploy

```bash
mlpal sites deploy ./dist --site my-site      # static: the built site
mlpal sites deploy . --site my-site           # static: a project root publishes its dist/, build/ or out/
mlpal sites deploy . --site my-api            # service: the project
```

A deploy uploads, is checked, and goes live. The site's kind decides how.

### Static sites

- **Build first.** A project root (one with `package.json`, `pyproject.toml` or `requirements.txt`) publishes only its `dist/`, `build/` or `out/`. That directory must contain `index.html`.
- **`index.html` at the root** of what you publish is required.
- **Published file types:** html, htm, css, js, mjs, json, map, webmanifest, txt, md, csv, xml, svg, png, jpg, jpeg, gif, webp, avif, ico, bmp, woff, woff2, ttf, otf, eot, mp3, ogg, wav, mp4, webm, pdf, wasm.
  - Anything else stops the deploy with `sites.file_type_not_allowed`, including extensionless files such as `CNAME`, `_redirects` or `LICENSE`.
  - Dotfiles and dot-directories are skipped, except `.well-known/`.
- **Limits:** 2,000 files, 25 MB per file, 100 MB per deploy.
- **Requests served:** only GET and HEAD. A static site has no forms backend; use a service site for that.
- `--no-promote` uploads and checks a deploy but keeps the current one live. Put it live later with `mlpal sites rollback <slug> <deploy-id>`.

### Service sites

**How it starts:**
- **Entry point:** the pod runs `npm start` if `package.json` has a start script, else `python app.py`, else `python main.py`, from the project root.
  - Watch out for a top-level `app/` directory: it becomes the directory the app runs from, so the usual `main.py` + `app/` package layout breaks.
- **Start your own server, on `$PORT` (8000) and all interfaces.** In Python the file must run it:
  ```python
  if __name__ == "__main__":
      uvicorn.run(app, host="0.0.0.0", port=int(os.environ.get("PORT", "8000")))
  ```
- **Answer `GET /health`** with any status below 500 within 30 s of starting, or the start fails.

**Dependencies:**
- **Runtime:** Python 3.13 or Node 22.
- **Python:** FastAPI, Starlette, uvicorn and aiohttp are preinstalled, so an app using only those needs no `pyproject.toml`. Anything else needs `pyproject.toml` **and** `uv.lock`.
  - `requirements.txt` is ignored.
  - Packages install from the lock with hash checks, wheels only: no build steps run.
  - Any `mlpal*` package must be locked from `https://pypi.mlpal.ai/simple/`.
- **Node:** `package.json` needs a `start` script and a `package-lock.json`. devDependencies aren't installed, and install scripts are skipped unless `package.json` has `"mlpal": {"installScripts": true}`.

**At runtime:**
- **Headers:** every request carries `X-Forwarded-Host`, the host the visitor used.
- **Limits:** request bodies are limited to 10 MB, and a request times out after 120 s.
- **Idle sleep:** a service sleeps when idle. The first request after that waits while it starts and installs its dependencies. That's about 10–15 s for a small app, longer with many dependencies.
- **Not reviewed:** service deploys go live when they finalize. They're never held for review, and `--no-promote` doesn't apply to them.

### Both kinds

- **Credential files** (`.env*`, `.npmrc`, keys, `.aws/`, `.ssh/`, …) are never uploaded.
- **Secret-looking files** stop the deploy: for example, a file containing a token such as a JWT, including a bundled "anon" key. Name it with `--allow-file <path>` if it's meant to be public. The path is relative to the directory being published (`assets/app.js`, not `dist/assets/app.js`).

### Exit codes

| Exit | Meaning |
|---|---|
| 0 | live (or ready, with `--no-promote`) |
| 3 | **held for review** (static only). The site is unchanged until a person approves it. Common causes: a password form posting to another domain, or a lot of HTML (over 300 HTML/SVG/XML files, or one over 2 MB). Don't redeploy to get around it. |
| 4 | refused for the account or payment: no invite, no verified card, wallet, or a suspended or held account |
| 2 | bad usage, e.g. a malformed slug or `--no-promote` on a service site |
| 69 | the service didn't answer; retry later |
| 77 | a delegated session expired; sign in again |
| 1 | anything else; the message says what |

## It's live, but the edge takes a moment

`edge: synced` means the change has been handed to the CDN. Visitors see it shortly after, usually within a minute. A new site can briefly answer "Site not found", and right after a redeploy some requests can still get the old version. Wait instead of redeploying:

```bash
mlpal sites status my-site --wait             # --timeout 60 by default
```

`--wait` exits 0 even if the edge is still catching up when it times out; check the `edge` field.

## Rollback, history, settings

```bash
mlpal sites deploys my-site                       # newest first, with scan verdicts
mlpal sites rollback my-site dpl_...              # put an earlier ready deploy live again (static only)
mlpal sites update my-site --spa                  # or --no-spa
mlpal sites status my-site
```

A service site is rolled back by deploying the earlier code again. The live deploy and the 4 before it are kept; older ones are purged.

## Custom domains

```bash
mlpal sites domains add my-site www.example.com   # prints the two DNS records to publish
mlpal sites domains verify my-site www.example.com
mlpal sites domains list my-site                  # pending_dns, issuing, active, failed, released
mlpal sites domains remove my-site www.example.com --yes
```

Publish exactly the two records `add` prints:
- a **TXT** record at `_mlpal-verify.<hostname>` proves you own the domain;
- a **CNAME** from the hostname to `domains.mlpal.app` routes it. An apex domain can use ALIAS, ANAME or CNAME flattening at the DNS host instead.

How it proceeds:
- **Pending:** a claim whose records aren't seen within **1 hour** lapses. Adding it again keeps the same records.
- **`verify`** checks right away. It exits 0 once the domain is issuing or active, and 1 while the records aren't seen yet, so scripts can poll on it.
- **Certificate:** once the records are seen, an HTTPS certificate is issued. It often takes minutes, and gives up after 72 hours. Then the domain is `active`.
- **TXT record:** keep it in place. The record is re-checked daily, and a domain whose TXT record stays gone for about 3 days is `released`.

Limits and refusals:
- 5 domains per site, 10 per account, 20 adds a day.
- `mlpal.ai` and `mlpal.app` names can't be attached, and a hostname another site holds is refused with `sites.domain_taken`.
- **Paused:** an add or a `verify` of a pending claim can answer `sites.custom_domains_paused`, meaning new domains are paused for now. Domains already active keep working; try again later.

## Usage and billing

```bash
mlpal sites usage my-site                 # this month: storage, egress and requests, in CU and USD
mlpal sites usage my-site --month 2026-11
```

Storage, egress and requests are billed to your wallet in compute units. A service site's run time is billed too, separately: it isn't in this output. Egress is metered from the CDN logs, so usage trails traffic by several minutes.

Each site has a monthly egress cap (100 GB by default). You're warned at 80%; past the cap, the site goes offline until the next month.

## Delete

```bash
mlpal sites delete my-site --yes
```

The site goes off the edge right away; visitors see "Site not found" within about a minute. A slug that ever served is never reissued.

## When a site is unavailable

| Visitor sees | Why |
|---|---|
| 404 "Site not found" | no such site; no live deploy yet (for example, the first deploy is held for review); or it was just created or deleted and the edge is catching up |
| 410 "disabled for violating the MLPal Acceptable Use Policy" | the site was taken down. Contact MLPal. |
| 503 "temporarily unavailable" | the account is on hold or suspended, or the site hit its egress cap |
| 405 / 400 (static) | a method other than GET or HEAD, or an invalid path |
| 402 / 503 (service) | the owner's wallet can't start it, or the app failed to start (see the `/health` rule above) |

## Scripting

- Most commands take `-o json`; `deploy`, `domains add` and `domains verify` take `--json`.
- `delete` and `domains remove` ask for confirmation; pass `--yes`.
- Use `--profile` to pick an account.
- A deploy never prints its upload URLs: they're credentials.
