# groceries-agent data repo

This is the **data + control plane** for a self-hosted [groceries-agent](https://github.com/caseyWebb/groceries-agent) instance — created from the [groceries-agent-data-template](https://github.com/caseyWebb/groceries-agent-data-template). It is **private** (it holds every member's personal state, your `wrangler.jsonc`, and your one deployment secret) and is read/written by the operator's grocery-mcp Worker via a GitHub App.

You do **not** fork the code repo. This repo is your control plane: deploy, onboarding, and revocation all run here, as thin callers of *reusable* workflows in the public code repo — so the code repo holds no secrets and you take updates by ref. Full operator setup: [docs/SELF_HOSTING.md](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SELF_HOSTING.md).

## Layout

```
wrangler.jsonc               # YOUR Worker config (overlaid onto the upstream source at deploy)
recipes/                     # shared recipe content (objective frontmatter + body)
aliases.toml  ingredients.toml  substitutions.toml   # shared reference data
skus/kroger.toml             # shared, location-tagged SKU cache
ready_to_eat/*.toml          # shared ready-to-eat catalogs
_indexes/                    # generated — do not hand-edit (CI rebuilds it)
users/
  <username>/                # one subtree per member (created lazily on first write)
    pantry.toml  preferences.toml  stockup.toml  grocery_list.toml
    taste.md  diet_principles.md  cooking_log.toml  meal_plan.toml  feeds.toml
    overlay.toml             # this member's rating/status per recipe
    notes/                   # this member's recipe notes
    substitutions.toml       # optional per-member substitution overrides
    recipes/                 # optional personal (unshared) recipes
.github/workflows/           # thin callers of the code repo's reusable workflows
```

## One-time setup

In **Settings → Secrets and variables → Actions**:

- Secret **`CLOUDFLARE_API_TOKEN`** — a Cloudflare token with Workers + KV edit. The *only* secret — this is why the repo is private.
- Variable **`TENANT_KV_ID`** — your `TENANT_KV` namespace id (Onboard/Revoke write KV by id).
- Variable **`WORKER_NAME`** (or **`WORKER_HOST`**) — lets Onboard show the connector URL in its summary.

Then fill in `wrangler.jsonc` (worker name, GitHub App id + installation, `DATA_*` coords, KV ids). See [SELF_HOSTING](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SELF_HOSTING.md) steps 5–6.

## Workflows — all run from **this repo's** Actions tab

| Workflow | Trigger | Does |
|---|---|---|
| **Deploy Worker** | manual | deploy the grocery-mcp Worker (overlays your `wrangler.jsonc` onto the upstream source) |
| **Onboard member** | manual | mint a member's invite code + allowlist entry in KV |
| **Revoke member** | manual | remove a member's allowlist entry + invite code |
| `build-indexes` | auto on recipe changes | regenerate `_indexes/` |
| `build-site` | auto on recipe changes | build + deploy the public cookbook to GitHub Pages (needs **GitHub Pro**; never reads `users/`) |

The build/deploy/provision **logic** lives in the code repo as reusable workflows; these are thin callers, so updates land centrally. Runs are billed to **this repo's owner**.

## Onboarding a member

Run the **Onboard member** Action from **this repo's Actions tab** — **not** a fork of the code repo (a fork of a public repo is itself public, so its Actions logs would leak the invite code). Enter the member's `username`; it mints their invite code (shown only in this **private** run's summary) and allowlists them in KV. Their `users/<username>/` subtree is **not** seeded — it's created automatically on their first write (e.g. setting their Kroger store).

Send the member their **invite code** + the **connector URL** (`https://<worker-host>/mcp`) + [`AGENT_INSTRUCTIONS.md`](https://github.com/caseyWebb/groceries-agent/blob/main/AGENT_INSTRUCTIONS.md). They need only a Claude.ai account and a Kroger account — no GitHub account. They add the connector in Claude.ai, enter the code at `/authorize`, then run their Kroger consent at `/oauth/init?tenant=<username>`. Remove someone later with **Revoke member**. Full flow: [docs/SELF_HOSTING.md](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SELF_HOSTING.md).
