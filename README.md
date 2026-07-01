# groceries-agent data repo

This is the **control plane and plugin marketplace** for a self-hosted [groceries-agent](https://github.com/caseyWebb/groceries-agent) instance — created from the [groceries-agent-data-template](https://github.com/caseyWebb/groceries-agent-data-template). It holds your `wrangler.jsonc`, the deploy workflow, and the **published plugin bundle** — and nothing else: **all data lives in Cloudflare**, not in this repo — your authored corpus (recipes + guidance markdown) in an **R2 bucket**, all operational and per-member state plus the derived recipe index in **D1**, and ephemeral infra in **KV**, all read and written by the operator's grocery-mcp Worker. The one Actions secret it carries (`CLOUDFLARE_API_TOKEN`) stays **encrypted** — so this repo is **public** (which is what lets members add your marketplace without a GitHub account), and member management is the Worker's Cloudflare Access-gated **`/admin`** panel, so no invite code is ever printed into a CI log.

You do **not** fork the code repo. This repo is your control plane: the deploy workflow runs here, as a thin caller of a *reusable* workflow in the public code repo — so the code repo holds no secrets and you take updates by ref. Full operator setup: [docs/SELF_HOSTING.md](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SELF_HOSTING.md).

## Layout

This repo holds your Worker config, the thin deploy workflow, and the plugin marketplace your deploy publishes. Your authored markdown (recipes + guidance) lives in **Cloudflare R2**; everything operational lives in **Cloudflare D1**.

```
wrangler.jsonc                   # YOUR Worker config (operator-owned keys; merged onto the upstream source at deploy)
.github/workflows/deploy.yml     # thin caller of the code repo's reusable deploy workflow
.claude-plugin/marketplace.json  # your plugin marketplace manifest (-> ./plugin/grocery-agent)
plugin/grocery-agent/            # the GENERATED plugin bundle — published by the deploy, your URL baked in (do not hand-edit)
docs/SCRAPER.md                  # operator guide for the optional off-cloud walled-source scraper
```

`plugin/grocery-agent/` is created by your **first deploy** (it builds the bundle with your connector URL and commits it here). Until then it doesn't exist — onboard members after your first deploy.

**Everything else is in D1, not this repo.** Each member's profile, pantry, meal plan, grocery list, cooking log, favorites/rejects, and notes — plus the shared corpus (aliases, SKU cache, stores, RSS feeds, the forwarded-newsletter discovery inbox) and the derived recipe index — all live in Cloudflare D1, isolated per tenant by a `tenant` column. There is no `users/` subtree and no per-member TOML; per-tenant state is created in D1 on each member's first write. (Data-model details: [docs/SCHEMAS.md](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SCHEMAS.md).)

## One-time setup

In **Settings -> Secrets and variables -> Actions**:

- Secret **`CLOUDFLARE_API_TOKEN`** — a Cloudflare token with Workers + KV + **D1** edit, used by the Deploy workflow. (D1 edit lets the deploy auto-provision the database and apply its schema migrations.)
- Secrets **`KROGER_CLIENT_ID`** + **`KROGER_CLIENT_SECRET`** (optional) — the deploy sets them as Worker secrets when present.
- Variable **`WORKER_HOST`** (e.g. `grocery-mcp.<you>.workers.dev`) — your connector host. The deploy bakes `https://<WORKER_HOST>/mcp` into your published plugin and stamps the README health badge. **Set it before deploying** — without it the deploy skips the plugin publish (you'd have no marketplace).

KV namespaces and the D1 database ship id-less and auto-provision on first deploy, pinning their ids back into `wrangler.jsonc` (the Deploy workflow has `contents: write` for this); the deploy also ensures the R2 corpus bucket. To enable the **`/admin`** panel, add a Cloudflare Access app on `<your-worker-host>/admin` and set `ACCESS_TEAM_DOMAIN` + `ACCESS_AUD` in `wrangler.jsonc` `vars` (non-secret identifiers). Also set **`ACCESS_ALLOWED_EMAILS`** as a **Worker secret** — *not* a committed var, since it's your email and this repo is public — for defense-in-depth. See [SELF_HOSTING](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SELF_HOSTING.md) steps 4–6.

**This repo is public.** Nothing here is secret: the `CLOUDFLARE_API_TOKEN` is an encrypted Actions secret (a visibility flip doesn't expose it), the `wrangler.jsonc` ids + `ACCESS_AUD` are non-secret identifiers, and recipes/member data live in R2/D1. Keep it that way — `.wrangler/` is gitignored (it caches your Cloudflare account id + email; never commit it).

## Workflows — run from **this repo's** Actions tab

| Workflow | Trigger | Does |
|---|---|---|
| **Deploy Worker** | manual | deploy the grocery-mcp Worker (overlays your `wrangler.jsonc`; auto-provisions KV + D1 and applies migrations), then build the plugin with your URL baked in and **publish it to this repo's marketplace** |

The Worker projects the recipe index from the R2 corpus on a schedule and serves the public cookbook at `/cookbook`. The build/deploy/provision **logic** lives in the code repo as a reusable workflow; this is a thin caller, so updates land centrally. Runs are billed to **this repo's owner**. Member management is **not** a workflow — it's the Worker's `/admin` panel (below).

Every run **rebuilds and republishes the plugin bundle** to this repo's marketplace — even when the only upstream change is personas/skills, e.g. the new **ingest skills** for the [walled-source scraper](#off-cloud-walled-source-scraper) (below). Upstream is now an aube workspace monorepo (`packages/{contract,worker,scraper}`), so its lockfile is **`aube-lock.yaml`** + `pnpm-workspace.yaml` (which replaced `package-lock.json`); this repo carries none of them — you take upstream by ref, not by vendoring it.

## Managing members — the `/admin` panel

Onboarding, revocation, and invite rotation live in the Worker's **`/admin`** panel (Cloudflare Access-gated), not a workflow — so minted invite codes never touch a CI log. Open `https://<your-worker-host>/admin`, complete the Access login, and **onboard** the member: enter their `username`; it allowlists them in `TENANT_KV` and shows their invite code **once** plus the connector URL. Their per-tenant state is created in D1 automatically on their first write.

Then get the agent into their Claude.ai: send them your **marketplace + their invite code** — `/plugin marketplace add <you>/groceries-agent-data`, then `/plugin install grocery-agent@groceries-agent-data`, then enter the code at `/authorize` and run Kroger consent at `/oauth/init?tenant=<username>`. No GitHub account needed (adding a public marketplace needs no auth), and updates you ship auto-pull. To remove or rotate a member, use the same `/admin` panel. Full flow: [docs/SELF_HOSTING.md](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SELF_HOSTING.md).

## Off-cloud walled-source scraper

Some recipe/newsletter sources sit behind a login or paywall the Cloudflare Worker can't reach. Upstream ships an **off-cloud scraper container** — a GHCR image you run on your own box (NAS, home server, occasionally-on laptop). It holds a session you capture for the paid site, pulls new content on a schedule, and pushes it to your Worker's **ingest** endpoint using an **ingest key** you mint in the `/admin` panel under **Config › Ingest Keys**. What it ingests lands in the same R2 corpus + D1 index everything else reads.

It's **optional operator infrastructure**: nothing about it lives in this repo, the Deploy workflow doesn't run it, and members never touch it — they just get the plugin's **ingest skills** in your published bundle. Operator quick-start (mint a key, capture a paid-site session, run the image via `docker run`/`docker-compose`): **[docs/SCRAPER.md](docs/SCRAPER.md)**. Upstream reference: [docs/SELF_HOSTING.md §"Walled-source scraper"](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SELF_HOSTING.md#walled-source-scraper) and [packages/scraper/README.md](https://github.com/caseyWebb/groceries-agent/blob/main/packages/scraper/README.md).
