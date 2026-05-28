# Fourier Language Documentation

Source for the Fourier smart-contract language documentation site.

| Domain | Source dir | Build |
|---|---|---|
| `fourier.fermi.world` | `docs/` | `mkdocs build` → `site/` |

Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and deployed via Vercel.

## Build locally

```bash
pip install -r requirements.txt
mkdocs serve --dev-addr 127.0.0.1:8001
```

## Deploy

Vercel project root: this directory. Build command and output directory are declared in [`vercel.json`](vercel.json).

The companion WaveLedger chain docs site lives in a separate repo: [DosseyRichards/FERMI-Documentation](https://github.com/DosseyRichards/FERMI-Documentation).

## Contributing

Open work items live in [TODO.md](TODO.md). When you write a page that references something not yet documented, append it there rather than writing a stub.
