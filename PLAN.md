# Tutorial Hub — Project Plan

**Goal:** A clean, modern, professional tutorial site at `https://ecellsworth.github.io`, built with Astro, with a visually stunning landing page and standard tutorial pages rendered from markdown files. Light and dark modes with a toggle. Hosted entirely on GitHub via GitHub Pages.

**Companion doc:** `HANDOFF.md` (implementation spec for whoever builds this).

---

## 1. Current state

- Repo: `ecellsworth/ecellsworth.github.io` (user Pages site — deploys to the root domain, no `base` path needed)
- Contents: README, LICENSE, `.gitignore`, and two existing tutorials:
  - `docs/tutorials/subagents-with-nvidia.md`
  - `docs/tutorials/prompt_runner.md`
- No site framework yet. GitHub Pages not yet configured for Actions deployment.

## 2. Decisions made

| Decision | Choice |
|---|---|
| Framework | Astro (latest stable, v6+), static output |
| Content location | Astro content collections in `src/content/tutorials/` (existing `docs/tutorials/*.md` migrated in with frontmatter added) |
| Theming | Light **and** dark mode, user toggle, respects `prefers-color-scheme`, persisted in `localStorage` |
| Visual assets | Tech-focused hero/OG imagery generated via connected MCP image-generation tools (spec in HANDOFF.md); SVG fallbacks acceptable |
| Deployment | GitHub Actions → GitHub Pages via official `withastro/action@v6` |
| Extra features | Theme toggle only for v1 (tags/search/TOC deferred) |

## 3. Milestones

**M1 — Scaffold & deploy pipeline (foundation)**
Initialize Astro project at repo root, configure `site: 'https://ecellsworth.github.io'`, add `.github/workflows/deploy.yml`, enable Pages (Source: GitHub Actions). Exit criteria: a "hello world" page live at ecellsworth.github.io.

**M2 — Content pipeline**
Define the `tutorials` content collection with typed frontmatter schema. Migrate the two existing markdown files, adding frontmatter. Dynamic route `src/pages/tutorials/[slug].astro` renders each tutorial. Exit criteria: both tutorials render at `/tutorials/<slug>/`.

**M3 — Design system & theming**
CSS custom-property token system (colors, spacing, type scale) with `[data-theme]` light/dark values, theme toggle component with no-flash inline script, base layout with header/footer/nav.

**M4 — Landing page**
Hero (generated art/gradient mesh, headline, CTA), tutorial card grid fed by the collection, polish (animations, responsive, accessible). This is the "visually stunning" milestone — detailed spec in HANDOFF.md §5.

**M5 — Tutorial page template**
Prose typography styles for rendered markdown (headings, code blocks with syntax highlighting, tables, images), metadata header (title, date, description), prev/next navigation, back-to-hub link.

**M6 — Assets, QA & launch**
Generate hero/OG images via MCP tools, favicon, meta/OG tags, Lighthouse pass (target ≥95 performance/accessibility), cross-browser check, final deploy, delete or redirect `docs/`.

## 4. Success criteria

- Landing page is distinctive and professional in both themes; no flash of wrong theme on load.
- Adding a new tutorial = drop a `.md` file with frontmatter into `src/content/tutorials/` and push. Nothing else.
- Fully static, no server; builds and deploys automatically on push to `main`.
- Lighthouse ≥95 on performance and accessibility.

## 5. Risks & notes

- **Theme flash (FOUC):** must set `data-theme` via inline `<head>` script before paint.
- **Frontmatter migration:** existing files have no frontmatter; schema should keep required fields minimal (`title`, `description`, `pubDate`) so migration is trivial.
- **MCP asset generation:** if image-gen MCP is unavailable at build time, fall back to CSS gradient mesh / inline SVG hero (the design works either way — see HANDOFF.md §6).
- **`docs/` folder:** GitHub Pages "deploy from /docs" is NOT used here; source switches to GitHub Actions. Keep `docs/tutorials/` until migration is verified, then remove.
