# groceries-agent data repo

This is the **data** for a self-hosted [groceries-agent](https://github.com/caseyWebb/groceries-agent) instance — created from the [groceries-agent-data-template](https://github.com/caseyWebb/groceries-agent-data-template). It is **private** (it holds every member's personal state) and is read/written by the operator's grocery-mcp Worker via a GitHub App.

## Layout

```
recipes/                     # shared recipe content (objective frontmatter + body)
aliases.toml  ingredients.toml  substitutions.toml   # shared reference data
skus/kroger.toml             # shared, location-tagged SKU cache
ready_to_eat/*.toml          # shared ready-to-eat catalogs
_indexes/                    # generated — do not hand-edit (CI rebuilds it)
users/
  <username>/                # one subtree per member (the operator provisions these)
    pantry.toml  preferences.toml  stockup.toml  grocery_list.toml
    taste.md  diet_principles.md  cooking_log.toml  meal_plan.toml  feeds.toml
    overlay.toml             # this member's rating/status per recipe
    notes/                   # this member's recipe notes
```

## CI

Two workflows call the code repo's **reusable workflows** (the build scripts live there, not here, so they update centrally):

- `build-indexes.yml` → regenerates `_indexes/` on recipe changes.
- `build-site.yml` → builds the public cookbook from `recipes/` and deploys it to this repo's **GitHub Pages** (needs **GitHub Pro** to publish public Pages from a private repo; the site never reads `users/`).

Runs are billed to **this repo's owner**, not the code repo's.

## Onboarding a member

The operator: create `users/<username>/` here (seed from the template stubs), add the member to the Worker's allowlist (`tenant:<username>` in KV), and hand them their invite code + connector URL. They need only a Claude.ai account and a Kroger account — no GitHub account.
