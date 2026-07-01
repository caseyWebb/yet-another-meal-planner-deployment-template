# Walled-source scraper (operator guide)

Some recipe and newsletter sources sit behind a login or paywall — a **walled
source** — that the grocery-mcp Worker can't reach. Cloudflare Workers have no
persistent browser session and can't hold your paid-site cookies, so upstream
ships a small **off-cloud scraper container** you run on your own machine (a
NAS, a home server, an occasionally-on laptop). It logs into the walled source
with a session you capture once, pulls new content on a schedule, and pushes it
to your Worker's **ingest** endpoint over HTTPS using an **ingest key** you
mint. It lands in the same R2 corpus + D1 index the rest of your agent reads.

The scraper is **optional operator infrastructure**. Nothing about it lives in
this repo, and it is not run by the Deploy workflow — you run it yourself,
outside Cloudflare. Members don't touch it: they get the plugin's **ingest
skills** automatically in your published bundle and drive it from Claude.ai.

This page is the operator quick-start. The authoritative reference is upstream:

- [docs/SELF_HOSTING.md → **"Walled-source scraper"**](https://github.com/caseyWebb/groceries-agent/blob/main/docs/SELF_HOSTING.md#walled-source-scraper)
- [packages/scraper/README.md](https://github.com/caseyWebb/groceries-agent/blob/main/packages/scraper/README.md) — full container reference
- [packages/scraper/scraper.example.toml](https://github.com/caseyWebb/groceries-agent/blob/main/packages/scraper/scraper.example.toml) — every config key, documented
- [packages/scraper/docker-compose.example.yml](https://github.com/caseyWebb/groceries-agent/blob/main/packages/scraper/docker-compose.example.yml) — the exact image tag, environment, and volume paths

> The snippets below show the **shape**. Copy the exact GHCR image reference and
> volume layout from `docker-compose.example.yml` upstream — that file is the
> source of truth. (GHCR namespaces are lowercase, so the image is
> `ghcr.io/caseywebb/...` even though the GitHub owner is `caseyWebb`.)

## 1. Mint an ingest key

The scraper authenticates to your Worker with an **ingest key** (a bearer
token). Mint one in the Worker's admin panel:

1. Open `https://<your-worker-host>/admin` and complete the Cloudflare Access login.
2. Go to **Config › Ingest Keys**.
3. Create a key and label it for the box/source it will run on. The key is shown
   **once** — copy it now.
4. You'll hand it to the container as the **`INGEST_API_KEY`** environment variable.

Rotate or revoke keys from the same **Config › Ingest Keys** screen when a box is
retired or a key leaks. The Worker validates every ingest request against the
minted keys, so revoking one immediately locks that scraper out.

## 2. Capture a paid-site session

The scraper needs an authenticated session for each walled source. Two ways
(both detailed in `packages/scraper/README.md`):

- **Interactive login** — run the container's login command; it drives a browser
  login flow where you sign in (and clear any 2FA) once, and the session is
  persisted to the mounted session/config volume.
- **Cookie import** — export your browser cookies for the site (e.g. with a
  `cookies.txt` extension) and drop the file into the config volume; the scraper
  loads them on start.

Captured sessions live on a mounted volume so they survive restarts. Treat the
session and cookie files like credentials — they grant access to your paid
account. Keep them on the host, never in this (public) repo.

## 3. Configure

Start from the upstream example config, then edit it:

```sh
mkdir -p scraper-config
curl -fsSL https://raw.githubusercontent.com/caseyWebb/groceries-agent/main/packages/scraper/scraper.example.toml \
  -o scraper-config/scraper.toml
```

At minimum, point it at your Worker's ingest URL — your connector host, e.g.
`https://<your-worker-host>` — and list the walled sources to pull.
`scraper.example.toml` documents every key.

## 4. Run the GHCR image

The container is published to GitHub Container Registry. **Confirm the exact tag
from `docker-compose.example.yml`**; it looks like
`ghcr.io/caseywebb/groceries-agent/scraper:latest`.

### docker run

```sh
docker run -d --name grocery-scraper \
  --restart unless-stopped \
  -e INGEST_API_KEY="<paste-the-key-you-minted>" \
  -v "$PWD/scraper-config:/config" \
  ghcr.io/caseywebb/groceries-agent/scraper:latest
```

- `-e INGEST_API_KEY` — the key from **Config › Ingest Keys** (step 1).
- `-v .../scraper-config:/config` — the mounted config volume holding
  `scraper.toml` plus your captured session/cookies. Use the **exact** container
  path (`/config` here) from the compose example.

### docker-compose

The ready-made compose file already wires the correct image, volumes, and
environment — download it and just supply your key:

```sh
curl -fsSL https://raw.githubusercontent.com/caseyWebb/groceries-agent/main/packages/scraper/docker-compose.example.yml \
  -o docker-compose.yml
```

Its shape (use the upstream file verbatim; set `INGEST_API_KEY` via a `.env`
file or your shell):

```yaml
services:
  scraper:
    image: ghcr.io/caseywebb/groceries-agent/scraper:latest
    restart: unless-stopped
    environment:
      INGEST_API_KEY: "${INGEST_API_KEY}"
    volumes:
      - ./scraper-config:/config
```

```sh
INGEST_API_KEY="<paste>" docker compose up -d
```

The container then pulls on its configured schedule and pushes new
recipes/newsletter items to your Worker's ingest endpoint, where they flow into
the same R2 corpus + D1 index everything else reads. Members see the results
through the agent's normal recipe/discovery flows, and the plugin's **ingest
skills** let them nudge it from Claude.ai — no operator action needed per pull.
