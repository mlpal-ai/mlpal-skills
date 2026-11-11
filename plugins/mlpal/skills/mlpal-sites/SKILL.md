---
name: mlpal-sites
description: Use when publishing a website or a small web app on MLPal Sites with the `mlpal sites` CLI. Covers invite-only access, creating a static or service site, deploying a built site or a FastAPI/Node app, review holds, rollbacks, custom domains, usage and billing, deleting, and scripting with exit codes.
---

# MLPal Sites

MLPal Sites hosts a site at `https://<slug>.mlpal.app`. A site is either:
- **static**: a built site (HTML, CSS, JS, images…), served from the CDN;
- **service**: a small server app (FastAPI, Flask, Node…), started on demand behind the same CDN.

Everything below is for `mlpal` **0.9.3 or later** (`mlpal --version`). See the `mlpal-cli` skill to install it and sign in.

## Access (invite-only for now)

```bash
mlpal sites access     # says whether Sites is open for this account
mlpal sites waitlist   # ask for an invite; safe to repeat, your first request keeps its place
```

While Sites is closed for the account, every other `sites` command says so. Don't retry in a loop.

## Create a site

```bash
mlpal sites create my-site                  # --kind auto: looks at the current directory
mlpal sites create my-api --dir ./api       # auto-detects from ./api
mlpal sites create docs --kind static --spa # --spa serves /index.html for extensionless paths
```

`--kind auto` picks **service** when the project has a server, and says why:
- `app.py` or `main.py` imports a web framework (FastAPI, Flask, Django, Starlette or aiohttp);
- `package.json` has a `start` script;
- a `Procfile`.

Anything else is **static**.

The slug:
- is 2–40 characters of `a-z`, `0-9` and `-`;
- is **unique forever**: a deleted site's slug is never reissued;
- can't contain well-known brand names.

An account has at most 3 sites, and at most 10 site creations and 10 deletions a day.

## Deploy

```bash
mlpal sites deploy ./dist --site my-site    # static: the built site
mlpal sites deploy . --site my-site         # static: a project root publishes its dist/, build/ or out/
mlpal sites deploy . --site my-api          # service: the project
```

A deploy uploads, is checked, and goes live. The site's kind decides how.

**Static sites:**
- **Build first.** A project root with `package.json` publishes only `dist/`, `build/` or `out/`, never its sources or `node_modules`. If there's no build output, the deploy stops and says so.
- **Published file types:** html, htm, css, js, mjs, json, map, webmanifest, txt, md, csv, xml, svg, png, jpg, jpeg, gif, webp, avif, ico, bmp, woff, woff2, ttf, otf, eot, mp3, ogg, wav, mp4, webm, pdf, wasm. Anything else stops the deploy with `sites.file_type_not_allowed`.
- **Limits:** 2,000 files, 25 MB per file, 100 MB per deploy.
- `--no-promote` uploads and checks a deploy but keeps the current one live. Put it live later with `mlpal sites rollback <slug> <deploy-id>`.

**Service sites:**
- The app listens on `$PORT` (8000), on all interfaces. For example, `uvicorn.run(app, host="0.0.0.0", port=int(os.environ.get("PORT", "8000")))`.
- **Python:** a project with dependencies needs `pyproject.toml` **and** `uv.lock`, so every package is checked against the lock's hashes; no build steps run. Any `mlpal*` package must be locked from `https://pypi.mlpal.ai/simple/`. A FastAPI or Starlette app needs no `pyproject.toml`: FastAPI, Starlette and uvicorn are already installed. Anything else (Flask, Django, …) needs one, with its lock.
- **Node:** `package.json` needs a `start` script and a `package-lock.json`. Install scripts are skipped unless `package.json` has `"mlpal": {"installScripts": true}`.
- **Every request carries `X-Forwarded-Host`:** the host the visitor used. Use it when building absolute URLs.
- **Idle sleep:** a service sleeps when idle. The first request after that waits for it to wake, usually a few seconds and up to about 15 s.
- **Payment:** service sites run on your wallet. A deploy refused for payment exits 4.

**For both kinds:** credential files (`.env*`, `.npmrc`, keys, `.aws/`, `.ssh/`, …) are never uploaded. A file that *looks* like it holds a secret stops the deploy unless you name it with `--allow-file <path>`.

**Exit codes for scripts:**

| Exit | Meaning |
|---|---|
| 0 | live (or ready, with `--no-promote`) |
| 3 | **held for review**: the content check flagged it, for example a password form posting to another domain. The site is unchanged until a person approves it. Don't redeploy to get around it. |
| 4 | refused for the account or payment (closed, suspended, card or balance) |
| 1 | anything else; the message says what |

## It's live, but the edge takes a moment

`edge: synced` means the change has been handed to the CDN. Visitors see it within about a minute: a new site can briefly answer "Site not found", and right after a redeploy some requests can still get the old version. Wait instead of redeploying:

```bash
mlpal sites status my-site --wait            # waits for the edge (default 60 s)
```

## Rollback, history, settings

```bash
mlpal sites deploys my-site                       # newest first, with scan verdicts
mlpal sites rollback my-site dpl_...              # put an earlier ready deploy live again (static)
mlpal sites update my-site --spa                  # or --no-spa
mlpal sites status my-site
```

A service site is rolled back by deploying the earlier code again. The last few deploys are kept; older ones are purged.

## Custom domains

```bash
mlpal sites domains add my-site www.example.com   # prints the two DNS records to publish
mlpal sites domains verify my-site www.example.com
mlpal sites domains list my-site                  # pending_dns, issuing, active, failed
mlpal sites domains remove my-site www.example.com
```

Publish exactly the two records `add` prints:
- a **TXT** record at `_mlpal-verify.<hostname>` proves you own it;
- a **CNAME** from the hostname to `domains.mlpal.app` routes it. An apex domain can use ALIAS, ANAME or CNAME flattening at the DNS host instead.

How it proceeds:
- **Pending:** a claim whose records aren't seen within **1 hour** lapses. Adding it again keeps the same records.
- **Certificate:** once the records are seen, the HTTPS certificate is issued, usually within minutes and up to 72 hours. Then the domain is `active`.
- **TXT record:** keep it in place. A domain whose TXT record stays gone for 3 days is released.
- **Limits:** 5 domains per site, 10 per account. `mlpal.ai` and `mlpal.app` names can't be attached.
- **Paused:** if adding answers `sites.custom_domains_paused`, new domains are paused for now. Domains already attached keep working; try again later.

## Usage and billing

```bash
mlpal sites usage my-site                 # this month: storage, egress, requests, in CU and USD
mlpal sites usage my-site --month 2026-11
```

Storage, egress and requests (plus service run time) are billed to your wallet in compute units. Egress is metered from the CDN logs, so usage trails traffic by several minutes.

Each site has a monthly egress cap. Past it, the site goes offline until the next month.

## Delete

```bash
mlpal sites delete my-site
```

The site goes off the edge at once (visitors see "Site not found" within about a minute). Its slug is never reissued.

## When a site is unavailable

| Visitor sees | Why |
|---|---|
| 404 "Site not found" | no such site, or it was just created or deleted, so the edge is still catching up |
| 410 "disabled for violating the Acceptable Use Policy" | the site was taken down. Contact MLPal. |
| 503 "temporarily unavailable" | the account is on hold or suspended, or the site hit its egress cap |

## Scripting

- Most commands take `-o json`; `deploy`, `domains add` and `domains verify` take `--json`.
- The exit codes above are stable.
- Use `--profile` to pick an account.
- A deploy prints no upload URLs: they're credentials.
