# groceries-agent data repo

This is the **data + control plane** for a self-hosted [groceries-agent](https://github.com/caseyWebb/groceries-agent) instance — created from the [groceries-agent-data-template](https://github.com/caseyWebb/groceries-agent-data-template). It is **private** (it holds every member's personal state, your `wrangler.jsonc`, and your one deployment secret) and is read/written by the operator's grocery-mcp Worker via a GitHub App.

You do **not** fork the code repo. This repo is your control plane: deploy, onboarding, and revocation all run here, as thin callers of *reusable* workflows in the public code repo — so the code repo holds no secrets and you take updates by ref. Full operator setup: [docs/SELF_HOSTING.md](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SELF_HOSTING.md).

## Layout

```
wrangler.jsonc               # YOUR Worker config (overlaid onto the upstream source at deploy)
recipes/                     # shared recipe content (objective frontmatter + body)
aliases.toml  substitutions.toml   # shared reference data
storage_guidance/            # shared curated put-away advice (class-keyed prose, read-only)
skus/kroger.toml             # shared, location-tagged SKU cache
feeds.toml  flyer_terms.toml                       # shared discovery feeds + flyer scan terms (curated)
discovery_sources.toml  discoveries_inbox.toml     # inbound-email allowlist (curated) + forwarded candidates (agent-written)
_indexes/                    # generated — do not hand-edit (CI rebuilds it)
users/
  <username>/                # one subtree per member (created lazily on first write)
    pantry.toml  preferences.toml  stockup.toml  grocery_list.toml
    ready_to_eat.toml        # this member's heat-and-eat catalog (per-tenant, not shared)
    kitchen.toml             # this member's owned cooking equipment (gates recipe makeability)
    taste.md  diet_principles.md  cooking_log.toml  meal_plan.toml  feeds.toml
    overlay.toml             # this member's rating/status per recipe
    notes/                   # this member's recipe notes
    substitutions.toml       # optional per-member substitution overrides
    recipes/                 # optional personal (unshared) recipes
.github/workflows/           # thin callers of the code repo's reusable workflows
```

## One-time setup

In **Settings → Secrets and variables → Actions**:

- Secret **`CLOUDFLARE_API_TOKEN`** — a Cloudflare token with Workers + KV edit, this is why the repo is private.
- Secrets **`KROGER_CLIENT_ID`** + **`KROGER_CLIENT_SECRET`** (optional) — the deploy sets them as Worker secrets when present.
- Variable **`WORKER_NAME`** (or **`WORKER_HOST`**) — optional; lets Onboard show the connector URL in its summary.

Then set **`GITHUB_APP_ID`** in `wrangler.jsonc` — that's the only value you fill in. The App private key goes in the Cloudflare dashboard (never a repo). See [SELF_HOSTING](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SELF_HOSTING.md) steps 4–5.

## Workflows — all run from **this repo's** Actions tab

| Workflow | Trigger | Does |
|---|---|---|
| **Deploy Worker** | manual | deploy the grocery-mcp Worker (overlays your `wrangler.jsonc` onto the upstream source) |
| **Onboard member** | manual | mint a member's invite code + allowlist entry in KV |
| **Revoke member** | manual | remove a member's allowlist entry + invite code |
| `build-indexes` | auto on recipe changes | regenerate `_indexes/` |
| `build-site` | auto on recipe changes | build + deploy the public cookbook to GitHub Pages (needs **GitHub Pro**; never reads `users/`) |
| **Build plugin** | manual | mint a plugin bundle with your Worker URL baked in, as a downloadable artifact to upload to claude.ai (build-only, no secrets) |

The build/deploy/provision **logic** lives in the code repo as reusable workflows; these are thin callers, so updates land centrally. Runs are billed to **this repo's owner**.

## Onboarding a member

Run the **Onboard member** Action from **this repo's Actions tab** — **not** a fork of the code repo (a fork of a public repo is itself public, so its Actions logs would leak the invite code). Enter the member's `username`; it mints their invite code (shown only in this **private** run's summary) and allowlists them in KV. Their `users/<username>/` subtree is **not** seeded — it's created automatically on their first write (e.g. setting their Kroger store).

Then get the agent into their Claude.ai. Easiest: run **Build plugin**, download the `.zip` artifact, and send them that file + their **invite code** — they upload it (Customize → upload a custom plugin file), open a fresh chat, enter the code at `/authorize`, then run Kroger consent at `/oauth/init?tenant=<username>`. No GitHub account needed. (Alternatives: your own marketplace if you forked, or just send the **connector URL** `https://<worker-host>/mcp` + [`AGENT_INSTRUCTIONS.md`](https://github.com/caseyWebb/groceries-agent/blob/main/AGENT_INSTRUCTIONS.md) for a Claude project.) Remove someone later with **Revoke member**. Full flow + the Worker-first update ordering: [docs/SELF_HOSTING.md](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SELF_HOSTING.md).
