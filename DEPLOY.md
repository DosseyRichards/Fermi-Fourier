# Deploying to Vercel

This site is built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and deployed as a static build on Vercel.

## One-time setup

1. Push this repo to GitHub (`DosseyRichards/Fermi-Fourier`).
2. In the Vercel dashboard, click **Add New → Project** and import the repo.
3. Choose framework preset **Other**. Vercel will read `vercel.json` for everything else.
4. Add an environment variable `PYTHON_VERSION = 3.12` if the default Python is older than 3.10.
5. Click **Deploy**.

## Domains

Bind this in **Project Settings → Domains**:

- `fourier.fermi.world` (primary)

Add a `CNAME` record at your DNS provider pointing `fourier.fermi.world` at `cname.vercel-dns.com`.

## Build configuration

Declared in [`vercel.json`](vercel.json). Equivalent to running:

```bash
pip install -r requirements.txt
mkdocs build --strict
```

`--strict` makes the build fail on any warning, so broken internal links surface immediately in CI rather than in production.

The output directory is `site/` (the MkDocs default).

## Local preview of the production build

```bash
pip install -r requirements.txt
mkdocs build --strict
python -m http.server 8001 --directory site
# open http://127.0.0.1:8001
```

## Triggering a redeploy

Push to `main` — Vercel rebuilds and promotes automatically. Pull requests get preview deployments.
