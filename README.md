# beastmode-docs

Public landing page + documentation for **BeastMode — Dark Factory**, the autonomous software pipeline by [develeap](https://develeap.com).

Live site: https://giladneiger.github.io/beastmode-docs/

The BeastMode source code is maintained in a separate private repository. This repo contains only the marketing site and public-facing documentation.

## Structure

- `index.html` — landing page (vanilla HTML/CSS/JS, dark-first)
- `_layouts/default.html` — Jekyll layout wrapping markdown docs with the same brand
- `_config.yml` — Jekyll config (plugins: `jekyll-optional-front-matter`, `jekyll-relative-links`)
- `*.md` — documentation pages, auto-rendered by Jekyll on GitHub Pages
- `assets/logo/` — brand SVG logos (icon, horizontal, dark/light variants, favicon)
- `assets/architecture-diagram.svg` — pipeline architecture overview (editable vector)

## Local preview

Requires Docker. Runs the same Jekyll build GitHub Pages uses:

```bash
docker run --rm --platform linux/amd64 \
  -v "$(pwd):/srv/jekyll" -p 4000:4000 \
  jekyll/jekyll:latest \
  sh -c "bundle install && jekyll serve --host 0.0.0.0"
```

Open http://localhost:4000 — site rebuilds on file changes.

## Deploy

Automatic: push to `main` → GitHub Pages builds + serves. No other infrastructure.

## License

MIT — see the main BeastMode repo.

---

Manufactured by [develeap](https://develeap.com).
