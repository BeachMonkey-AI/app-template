# app-template

Template for Bob's apps. **Copied, never linked** — no shared package, no submodule. An app that
outgrows this template edits its own files and nothing breaks elsewhere.

## Using this template

1. `gh repo create beachmonkey-ai/<slug> --public --template BeachMonkey-AI/app-template`
2. Fill in `index.html`'s title/description, `public/manifest.webmanifest`, `explore.json`, and
   `CLAUDE.md`.
3. Replace `public/icon.svg` with the app's real icon, then `npm run gen-icons`.
4. Add the two caller workflows below to the new repo's own `.github/workflows/`.
5. Build the app in `src/` (or replace the vanilla scaffold with a Vite one — see below).
6. Write the app's own `README.md` in the shape described at the bottom of this file.

## Directory structure

```
index.html                  # <100 lines; tokens linked, PWA wired, all paths relative
src/{main.js, app.css}      # the app itself
styles/tokens.css           # design tokens (below) — copy, override --accent per app
public/manifest.webmanifest, sw.js, icon.svg
scripts/{build.mjs, gen-icons.mjs}
.github/workflows/{ci.yml, pages.yml}   # workflow_call reusables — see caller snippet below
explore.json                 # app metadata Bob copies into explore's apps.json when listing
CLAUDE.md, spec.md, README.md
```

## Design tokens

`styles/tokens.css` — derived from Explore's palette. **`--accent`/`--accent-dim` are the expected
per-app override point.** Full overrides (a completely different palette) are legitimate — just
note it in the app's own `CLAUDE.md` under "Token deviations."

## The base-path problem, solved by not having one

Pages previews live at `/preview/pr-<N>/` while production lives at `/`. A naive build would need
to know which one it's building for and rewrite every absolute path (`start_url`, icon `src`,
`sw.js` registration) accordingly — easy to get wrong, and exactly the kind of thing that breaks
PWA installability on preview only, silently.

**This template avoids that entirely: every path is relative, never root-absolute.**
- `index.html` links `styles/tokens.css`, not `/styles/tokens.css`.
- `manifest.webmanifest`'s `start_url`/`scope` are `"."`, not `"/"` — per the Web App Manifest
  spec, these resolve relative to the *manifest's own URL*, not the page's.
- `sw.js` resolves its own shell-cache list against `self.location` (its own URL), not a
  hardcoded root.

Because of this, `scripts/build.mjs` takes **no base-path parameter** — it just copies files into
`dist/` and bumps the service-worker cache name from a content hash. The same `dist/` output is
correct whether it's deployed at `/` or `/preview/pr-42/`.

## Icons

`public/icons/` is generated, not committed (gitignored) — CI runs `npm run gen-icons` before
every build. One source SVG (`public/icon.svg`) → four PNGs (uses `sharp`):
`icon-192.png`, `icon-512.png`, `icon-maskable.png` (content shrunk to fit the ~40%-radius safe
zone, background-filled — see `scripts/gen-icons.mjs` for why), `apple-touch-icon.png` (flattened,
no transparency — iOS renders alpha as black).

## CI and deploy — the caller workflow

`ci.yml` and `pages.yml` in this template are **reusable workflows** (`workflow_call`) — they run
in whichever repo calls them. Each app repo needs its own thin caller, e.g.
`.github/workflows/deploy.yml`:

```yaml
name: Deploy
on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches: [main]
jobs:
  ci:
    uses: BeachMonkey-AI/app-template/.github/workflows/ci.yml@main
  pages:
    uses: BeachMonkey-AI/app-template/.github/workflows/pages.yml@main
```

`pages.yml` deploys `main` pushes to `gh-pages:/` (production) and each open PR to
`gh-pages:/preview/pr-<N>/`, cleaning that directory up when the PR closes. Set the repo's Pages
source to the `gh-pages` branch once, after the first deploy creates it.

`ci.yml` is deliberately light — build-check only, for visibility on apps that currently have no
CI at all. Not a test suite.

## Two scaffold variants

**Vanilla (default)** — what's in this template as shipped: a single `index.html`, CDN deps if
needed, no bundler. Matches three of the four existing apps.

**Vite** — for apps that outgrow vanilla (React/Tailwind → `dist/`). Swap `src/main.js` for a Vite
project; set `base: './'` in `vite.config.js` (Vite's own relative-base mode) to keep the same
base-path-free deploy story above. Not scaffolded here yet — do this by hand until a second app
needs it.

## Local dev

```
npm install
npm run gen-icons   # regenerates public/icons/ — gitignored, not committed
npm run build        # writes dist/
```

No dev server is bundled — open `index.html` directly, or serve `.` with any static file server.

---

## The README shape every app built from this template should follow

*(This section documents the shape — this file itself, being the template's own README, doesn't
follow it. Delete this section and everything above it when writing a real app's README, and
replace with the shape below.)*

```markdown
# <App Name>

<One-line pitch.>

**Live:** <url>

## What it does

<a few sentences>

## Model

vanilla | vite

## Local dev

<how to run it locally>

## Token deviations

<e.g. "--accent overridden to #38bdf8 (neon cyan)"; or "none">
```
