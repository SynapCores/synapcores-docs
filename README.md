# synapcores-docs

Source for [docs.synapcores.com](https://docs.synapcores.com).

Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/);
deployed to GitHub Pages on every push to `main` via
[`.github/workflows/deploy.yml`](.github/workflows/deploy.yml).

## Local preview

```bash
pip install mkdocs-material mkdocs-material-extensions pymdown-extensions
mkdocs serve
# → http://127.0.0.1:8000
```

## Editing

Content lives under `docs/`. The sidebar is configured in `mkdocs.yml`'s
`nav:` block. Page titles come from the first H1 in each markdown file.

## Structure

```
docs/
├── index.md           # Landing page
├── quickstart.md      # Install + first run
├── requirements.md    # Hardware/platform support
├── ai-setup.md        # Wiring up local GGUF / Ollama / OpenAI
├── limits.md          # CE quotas + caps
├── enterprise.md      # CE vs EE feature comparison
└── support.md         # Bug reports, security disclosure
```

## Deploy

Push to `main`. The `Build and deploy docs` workflow:

1. Installs MkDocs Material on the runner
2. Runs `mkdocs build --strict` (any broken link/syntax fails the build)
3. Uploads the `_site/` artifact to GitHub Pages
4. Promotes the artifact via `actions/deploy-pages@v4`

GitHub then serves at `docs.synapcores.com` (CNAME in this repo's root).
