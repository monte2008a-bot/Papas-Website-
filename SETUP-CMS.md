# Papas — Content Manager (Sveltia CMS) setup

The shop owner can edit the website at **`/admin`** — no code, no GitHub knowledge
needed. They log in with GitHub, change text / prices / hours, hit **Publish**, and
the live site updates automatically a minute later (GitHub Pages rebuild).

This file documents the **one-time setup** you (the developer) need to do once, plus
how everything fits together.

---

## How it works

- **`/admin/index.html`** loads [Sveltia CMS](https://github.com/sveltia/sveltia-cms)
  from a CDN. (Sveltia is a faster, maintained drop-in successor to Decap / Netlify CMS.)
- **`/admin/config.yml`** defines what's editable and where it's stored.
- Editable content lives in **`/content/*.json`** (one file per homepage section).
- **`index.html`** fetches those JSON files on load and renders the sections from them.
  The hand-coded markup is kept as a **fallback** — if a JSON fetch ever fails, the
  site still renders its built-in content. The motion/animation layer only starts
  *after* content is rendered, so scroll reveals keep working.

```
content/hero.json        → Hero tagline + buttons
content/counter.json     → "The Counter" sandwiches (items, prices, tags)
content/bowls.json       → "The Bowls" (items, prices, tags)
content/locations.json   → Locations cards
content/about.json       → About copy + facts
content/reviews.json     → Customer reviews
content/info.json        → Hours / address / phone columns
```

> Note: the decorative photo placeholders (the About gallery tiles and the menu
> "image style" swatches) are design elements, not real photos yet. When real photos
> exist, wire them in and they can be added to the CMS as image fields.

---

## One-time setup: GitHub login for a non-technical owner

Because the site is static (GitHub Pages, no server), GitHub OAuth needs a tiny
auth broker. Sveltia provides a free Cloudflare Worker for exactly this:
**`sveltia-cms-auth`**. You set it up once.

### 1. Create a GitHub OAuth App
1. Go to **https://github.com/settings/applications/new**
2. Fill in:
   - **Application name:** `Papas CMS`
   - **Homepage URL:** `https://monte2008a-bot.github.io/Papas-Website-/`
   - **Authorization callback URL:** *(leave for a moment — you'll paste the worker
     URL here after step 2, as `<WORKER_URL>/callback`)*
3. Click **Register application**. Copy the **Client ID**.
4. Click **Generate a new client secret**. Copy the **Client secret** (shown once).

### 2. Deploy the auth worker to Cloudflare
1. Create a free **Cloudflare** account (https://dash.cloudflare.com/sign-up).
2. Deploy the worker — easiest is the one-click button in the
   **[sveltia-cms-auth README](https://github.com/sveltia/sveltia-cms-auth)**
   ("Deploy to Cloudflare Workers"), or run `wrangler deploy` locally from a clone.
3. After deploy you get a URL like:
   `https://sveltia-cms-auth.<your-subdomain>.workers.dev`
   — **copy it**.

### 3. Connect the two
1. Back in the **GitHub OAuth App** (step 1), set the **Authorization callback URL** to:
   ```
   https://sveltia-cms-auth.<your-subdomain>.workers.dev/callback
   ```
   and save.
2. In the **Cloudflare** dashboard → your worker → **Settings → Variables**, add:
   | Variable | Value |
   |---|---|
   | `GITHUB_CLIENT_ID` | the Client ID from step 1 |
   | `GITHUB_CLIENT_SECRET` | the Client secret from step 1 (mark as **encrypted/secret**) |
   | `ALLOWED_DOMAINS` | `monte2008a-bot.github.io` *(optional, recommended — locks the broker to this site)* |
3. Redeploy / save the worker.

### 4. Point the CMS at the worker
Edit **`admin/config.yml`** and replace the placeholder `base_url` with your worker URL:
```yaml
backend:
  name: github
  repo: monte2008a-bot/Papas-Website-
  branch: main
  base_url: https://sveltia-cms-auth.<your-subdomain>.workers.dev
```
Commit + push to `main`.

### 5. Give the owner access
The owner needs **write access** to the `monte2008a-bot/Papas-Website-` repo:
- Repo → **Settings → Collaborators → Add people** → invite their GitHub account.

Done. They visit **`https://monte2008a-bot.github.io/Papas-Website-/admin/`**, click
**Login with GitHub**, and start editing.

---

## Editing without the worker (developer / quick test)

Sveltia also supports signing in with a **Personal Access Token** — no worker needed,
but only practical for technical users. If `base_url` is not set / not working, Sveltia
offers a token sign-in option in the login dialog. Create a fine-grained PAT with
**Contents: read/write** on this repo and paste it in.

---

## Local preview

Because the page fetches JSON, open it through a server (not `file://`):
```bash
python3 -m http.server 8000
# then visit http://localhost:8000/
```
The `/admin` UI needs the deployed worker (or PAT) to authenticate against GitHub, so
it's normal for login to only work on the live GitHub Pages URL.

---

## Branches

The CMS commits to **`main`** (the published branch). Ongoing developer work happens
on feature branches and is merged to `main` as usual.
