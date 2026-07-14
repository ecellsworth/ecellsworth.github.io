# Tutorial Hub — Implementation Handoff

Spec for building the site described in `PLAN.md`. Written for an implementing agent or developer starting from this repo.

---

## 1. Stack & constraints

- **Astro latest stable** (`npm create astro@latest -- --template minimal --typescript strict`), static output (default). No adapter needed.
- Plain Astro components + scoped CSS / global tokens. No UI framework required.
- Node 20+ locally; CI uses the action default (Node 24). **Commit `package-lock.json`** — the deploy action detects the package manager from the lockfile.
- Repo is a *user* Pages site (`ecellsworth.github.io`) → deploys at the domain root. **Set `site`, do NOT set `base`.**

`astro.config.mjs`:
```js
import { defineConfig } from 'astro/config';

export default defineConfig({
  site: 'https://ecellsworth.github.io',
});
```

## 2. Project structure

```
/
├── .github/workflows/deploy.yml
├── astro.config.mjs
├── package.json
├── public/
│   ├── favicon.svg
│   └── og-default.png            # generated, see §6
├── src/
│   ├── content.config.ts         # collection definition
│   ├── content/
│   │   └── tutorials/
│   │       ├── subagents-with-nvidia.md   # migrated from docs/tutorials/
│   │       └── prompt-runner.md           # migrated (rename: kebab-case slugs)
│   ├── layouts/
│   │   ├── BaseLayout.astro      # <head>, theme script, header, footer
│   │   └── TutorialLayout.astro  # prose wrapper, metadata header, prev/next
│   ├── components/
│   │   ├── Header.astro
│   │   ├── Footer.astro
│   │   ├── ThemeToggle.astro
│   │   ├── Hero.astro
│   │   └── TutorialCard.astro
│   ├── pages/
│   │   ├── index.astro           # landing page
│   │   └── tutorials/
│   │       ├── index.astro       # all-tutorials grid (optional but cheap)
│   │       └── [slug].astro      # tutorial renderer
│   └── styles/
│       ├── tokens.css            # design tokens, both themes
│       └── global.css            # reset, base typography, prose styles
```

## 3. Content collection

`src/content.config.ts`:
```ts
import { defineCollection, z } from 'astro:content';
import { glob } from 'astro/loaders';

const tutorials = defineCollection({
  loader: glob({ pattern: '**/*.md', base: './src/content/tutorials' }),
  schema: z.object({
    title: z.string(),
    description: z.string(),
    pubDate: z.coerce.date(),
    updatedDate: z.coerce.date().optional(),
    difficulty: z.enum(['beginner', 'intermediate', 'advanced']).optional(),
    heroImage: z.string().optional(),
    draft: z.boolean().default(false),
  }),
});

export const collections = { tutorials };
```

**Migration:** move `docs/tutorials/*.md` → `src/content/tutorials/`, rename `prompt_runner.md` → `prompt-runner.md`, and prepend frontmatter to each (derive `title`/`description` from the file's content; use the git file date for `pubDate`). Filter `draft: true` out of all listings and `getStaticPaths`. Verify both render, then delete `docs/`.

**Adding a tutorial later** (this is the whole authoring workflow — keep it this simple):
```md
---
title: "My New Tutorial"
description: "One-sentence summary shown on cards and in meta tags."
pubDate: 2026-07-13
---
Markdown body...
```

## 4. Theming (light + dark)

- Tokens as CSS custom properties in `tokens.css`, scoped `:root[data-theme='light']` / `:root[data-theme='dark']`.
- **No-flash script** — inline in `<head>` of `BaseLayout.astro`, before any CSS paint:
```html
<script is:inline>
  const t = localStorage.getItem('theme')
    ?? (matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light');
  document.documentElement.dataset.theme = t;
</script>
```
- `ThemeToggle.astro`: sun/moon icon button in the header; on click flips `data-theme`, writes `localStorage.theme`. Give it `aria-label="Toggle color theme"`. Animate the icon swap subtly (~150 ms).
- Suggested palette (adjust to taste, keep WCAG AA contrast):

| Token | Light | Dark |
|---|---|---|
| `--bg` | `#fafafa` | `#0b0e14` |
| `--surface` | `#ffffff` | `#141821` |
| `--text` | `#1a1d24` | `#e6e9ef` |
| `--text-muted` | `#5c6370` | `#9aa3b2` |
| `--accent` | `#4f46e5` | `#818cf8` |
| `--accent-2` | `#06b6d4` | `#22d3ee` |
| `--border` | `#e5e7eb` | `#252b38` |

- Typography: Inter (or system stack) for UI/body, `JetBrains Mono` for code. Self-host via `@fontsource` packages or Astro's fonts API — no external font CDN calls.

## 5. Landing page spec (the "visually stunning" part)

Sections, top to bottom:

1. **Header** — sticky, translucent (`backdrop-filter: blur`), logo/wordmark "Tutorial Hub", nav (Tutorials), theme toggle.
2. **Hero** — full-viewport-ish (~80vh). Headline (e.g., "Learn by building.") with a gradient text accent, one-line subhead, primary CTA button → `/tutorials/`. Background: generated tech-themed artwork (§6) OR layered CSS gradient mesh + subtle animated grid/particle SVG. Must look intentional in both themes (either theme-specific images or an overlay that adapts via tokens).
3. **Tutorial grid** — "Latest tutorials", responsive card grid (1/2/3 columns), fed by `getCollection('tutorials')` sorted by `pubDate` desc. Cards: title, description, date, difficulty badge if present; hover lift (`transform: translateY(-4px)` + shadow, ~200 ms ease).
4. **Footer** — minimal: copyright, GitHub link.

Polish requirements: scroll-reveal fade-ups via `IntersectionObserver` (respect `prefers-reduced-motion`), consistent spacing scale, max content width ~72rem, mobile-first responsive.

## 6. Asset generation (via MCP tools)

Generate with a connected image-generation MCP server (e.g., Higgsfield `generate_image`); if unavailable, use the CSS/SVG fallback and note it.

| Asset | Spec | Prompt direction |
|---|---|---|
| Hero background (dark) | 2400×1350 WebP, <300 KB after compression | "Abstract dark tech background, deep navy, glowing indigo and cyan gradient mesh, subtle circuit-like line network, soft glow, minimalist, professional, no text" |
| Hero background (light) | same | Same concept on near-white background, soft pastel indigo/cyan |
| OG image (`public/og-default.png`) | 1200×630 PNG | Same visual language + "Tutorial Hub" wordmark, high contrast |
| Favicon | hand-authored inline SVG | Simple geometric mark using `--accent`, works at 16px |

Compress rasters (e.g., `squoosh`/`sharp`) and serve through Astro's `<Image>` component where possible.

## 7. Tutorial page template

- `TutorialLayout.astro` wraps `render(entry)` output (Astro 5+ content API: `import { render } from 'astro:content'`).
- Metadata header: title (h1), description, pub date, difficulty badge.
- Prose styles (scoped `.prose` class in `global.css`): 65–75ch measure, 1.7 line-height, styled `h2/h3` with anchor links, blockquotes, tables, images (rounded, max-width 100%).
- **Code blocks:** Astro's built-in Shiki. Configure dual themes so highlighting follows the site theme:
```js
// astro.config.mjs
markdown: {
  shikiConfig: { themes: { light: 'github-light', dark: 'github-dark' } },
}
```
plus the documented CSS to switch `.shiki` colors on `[data-theme='dark']`. Add a copy-to-clipboard button on code blocks (small inline script).
- Prev/next links between tutorials (sorted by `pubDate`), and "← All tutorials".
- Per-page meta: `<title>`, description, `og:image` (fallback `og-default.png`).

## 8. Deployment

`.github/workflows/deploy.yml` (this is the current official recipe — verified July 2026):
```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: withastro/action@v6

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```

One-time manual step: repo **Settings → Pages → Source: GitHub Actions**.

## 9. Acceptance checklist

- [ ] `npm run build` succeeds with zero warnings; site deploys on push to `main`
- [ ] Landing page renders correctly in both themes; toggle persists across reloads; no theme flash
- [ ] Both migrated tutorials render at `/tutorials/subagents-with-nvidia/` and `/tutorials/prompt-runner/`
- [ ] New `.md` with frontmatter appears on landing grid automatically
- [ ] Code blocks syntax-highlighted in both themes; copy button works
- [ ] Lighthouse ≥95 performance & accessibility (mobile)
- [ ] Keyboard navigable; visible focus states; `prefers-reduced-motion` respected
- [ ] OG/meta tags present; favicon works; 404 page exists (`src/pages/404.astro`)
- [ ] `docs/` removed after migration verified

## 10. Out of scope (v1)

Tags/filtering, client-side search (Pagefind), sidebar TOC, RSS, comments, analytics. All are clean additions later; the content collection schema above already leaves room for them.
