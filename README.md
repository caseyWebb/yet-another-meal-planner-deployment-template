# groceries-agent data repo

This is the **control plane** for a self-hosted [groceries-agent](https://github.com/caseyWebb/groceries-agent) instance — created from the [groceries-agent-data-template](https://github.com/caseyWebb/groceries-agent-data-template). It holds your `wrangler.jsonc` and the deploy/build workflows, and nothing else: **all data lives in Cloudflare**, not in this repo — your authored corpus (recipes + guidance markdown) in an **R2 bucket**, all operational and per-member state plus the derived recipe index in **D1**, and ephemeral infra in **KV**, all read and written by the operator's grocery-mcp Worker. The one Actions secret it carries (`CLOUDFLARE_API_TOKEN`) stays **encrypted** — member state never lives here, and member management is the Worker's Cloudflare Access-gated **`/admin`** panel, so no invite code is ever printed into a CI log.

You do **not** fork the code repo. This repo is your control plane: the deploy + build workflows run here, as thin callers of *reusable* workflows in the public code repo — so the code repo holds no secrets and you take updates by ref. Full operator setup: [docs/SELF_HOSTING.md](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SELF_HOSTING.md).

## Layout

This repo holds only your Worker config and the thin deploy/build workflows. Your authored markdown (recipes + guidance) lives in **Cloudflare R2**; everything operational lives in **Cloudflare D1**.

```
wrangler.jsonc               # YOUR Worker config (operator-owned keys; merged onto the upstream source at deploy)
.github/workflows/           # thin callers of the code repo's reusable workflows
```

**Everything else is in D1, not this repo.** Each member's profile, pantry, meal plan, grocery list, cooking log, favorites/rejects, and notes — plus the shared corpus (aliases, SKU cache, stores, RSS feeds, the forwarded-newsletter discovery inbox) and the derived recipe index — all live in Cloudflare D1, isolated per tenant by a `tenant` column. There is no `users/` subtree and no per-member TOML; per-tenant state is created in D1 on each member's first write. (Data-model details: [docs/SCHEMAS.md](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SCHEMAS.md).)

## One-time setup

In **Settings → Secrets and variables → Actions**:

- Secret **`CLOUDFLARE_API_TOKEN`** — a Cloudflare token with Workers + KV + **D1** edit, used by the Deploy workflow. (D1 edit lets the deploy auto-provision the database and apply its schema migrations.)
- Secrets **`KROGER_CLIENT_ID`** + **`KROGER_CLIENT_SECRET`** (optional) — the deploy sets them as Worker secrets when present.
- Variable **`WORKER_HOST`** (or **`WORKER_NAME`**) — optional; lets the deploy stamp the README health badge and resolve the connector host.

KV namespaces and the D1 database ship id-less and auto-provision on first deploy, pinning their ids back into `wrangler.jsonc` (the Deploy workflow has `contents: write` for this); the deploy also ensures the R2 corpus bucket. To enable the **`/admin`** panel, add a Cloudflare Access app on `<your-worker-host>/admin` and set `ACCESS_TEAM_DOMAIN` + `ACCESS_AUD` in `wrangler.jsonc` `vars`. See [SELF_HOSTING](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SELF_HOSTING.md) steps 4–6.

## Workflows — all run from **this repo's** Actions tab

| Workflow | Trigger | Does |
|---|---|---|
| **Deploy Worker** | manual | deploy the grocery-mcp Worker (overlays your `wrangler.jsonc` onto the upstream source; auto-provisions KV + D1 and applies migrations) |
| **Build plugin** | manual | mint a plugin bundle with your Worker URL baked in, as a downloadable artifact to upload to claude.ai (build-only, no secrets) |

The Worker projects the recipe index from the R2 corpus on a schedule and serves the public cookbook at `/cookbook`. The build/deploy/provision **logic** lives in the code repo as reusable workflows; these are thin callers, so updates land centrally. Runs are billed to **this repo's owner**. Member management is **not** a workflow — it's the Worker's `/admin` panel (below).

## Managing members — the `/admin` panel

Onboarding, revocation, and invite rotation live in the Worker's **`/admin`** panel (Cloudflare Access-gated), not a workflow — so minted invite codes never touch a CI log. Open `https://<your-worker-host>/admin`, complete the Access login, and **onboard** the member: enter their `username`; it allowlists them in `TENANT_KV` and shows their invite code **once** plus the connector URL. Their per-tenant state is created in D1 automatically on their first write.

Then get the agent into their Claude.ai. Easiest: run **Build plugin**, download the `.zip` artifact, and send them that file + their **invite code** — they upload it (Customize → upload a custom plugin file), open a fresh chat, enter the code at `/authorize`, then run Kroger consent at `/oauth/init?tenant=<username>`. No GitHub account needed. (Alternatives: your own marketplace if you forked, or just send the **connector URL** `https://<worker-host>/mcp` + [`AGENT_INSTRUCTIONS.md`](https://github.com/caseyWebb/groceries-agent/blob/main/AGENT_INSTRUCTIONS.md) for a Claude project.) To remove or rotate a member, use the same `/admin` panel. Full flow + the Worker-first update ordering: [docs/SELF_HOSTING.md](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SELF_HOSTING.md).
