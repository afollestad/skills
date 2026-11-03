---
name: static-site-rules
description: Build or change a hand-written static website (HTML, CSS, and vanilla JavaScript on GitHub Pages) with consistent theming, components, accessibility, assets, and publishing. Use when creating a static site, adding pages or sections, changing colors or shared components, or making icons, screenshots, or link previews.
argument-hint: "[site or change]"
effort: high
---

Sites are hand-written HTML, CSS, and vanilla JavaScript on GitHub Pages: no build step, frameworks, CSS libraries, or web fonts, and no dependencies beyond unmodified vendored code. Sibling sites share one base, and each keeps its own palette and motif.

Read the repo's `AGENTS.md` first; it wins where it disagrees with this skill. The exact values, class names, and file contracts are in [`references/design-system.md`](references/design-system.md) and [`references/code.md`](references/code.md). For anything not covered here, copy from a sibling site that already does something similar rather than inventing, but where a sibling breaks a rule below, follow the rule.

## Rules

- **Three sheets.** `styles/index-common.css` holds everything theme-independent; `index-light.css` and `index-dark.css` hold tokens and theme-painted backgrounds. Both theme sheets ship on every load, media-scoped to `prefers-color-scheme`, so the page follows the OS with no JavaScript. `scripts/dark-mode.js` acts only on a stored override.
- **Anything else themed** ships both variants, media-scoped, with `data-theme` for the script to flip (`<picture>` with a dark `<source>`). Never swap `src` from script or hide a duplicate with `display`.
- **Colors come from the product** (its icon, logo, or own UI), with the source named in a comment, and stay distinct from sibling sites' palettes.
- **Contrast.** Text holds 4.5:1 on whatever it sits on, including `--bg`, the glow's strongest point, and chips; the title gradient and non-text UI hold 3:1. Note checked ratios in the tokens' comments. Composite translucent layers over what they sit on before measuring.
- **Shared brand mark.** When sites share a logo, each loads it from its one canonical URL rather than a copy, uncropped on a theme-aware backing (`--logo-circle`) that holds 3:1 with its main colors, with the same values on every site.
- **Scroll-to-top** is `overview-return`. Ad blockers hide `back-to-top`, `jump-to-top`, and `scroll-to-top`.
- **Accessibility.** Hover styles only inside `@media (hover: hover) and (pointer: fine)`; a 2px `:focus-visible` ring; `prefers-reduced-motion` honored in CSS and in scripted scrolling; no `user-scalable=no`; `aria-label` on icon-only controls and `aria-hidden="true"` on their SVGs.
- **Images.** Every published `<img>` has `width`, `height`, and `decoding="async"`, plus `fetchpriority="high"` for the hero or `loading="lazy"` below the fold. Decorative ones get `alt=""`.
- **Publishing.** Pages runs Jekyll on the branch root and copies files without front matter verbatim, so add none. List every root file that isn't part of the site in `_config.yml`'s `exclude` by name, since `**/*.md` misses the root. Publish third-party licenses that require it as `.txt`, so the Markdown exclusions can't drop them.
- **Link previews** are 1200x630 opaque PNGs on the dark theme, rendered with headless Chrome from a committed, unpublished `og-image/`. Re-render in the same change as anything they show.
- **Shared patterns stay in sync.** When one changes on a site, carry it to every sibling that has it, and update this skill if a rule here changed.

## Update together

| Change | Also update |
|---|---|
| A theme's `--bg` | `theme-color` metas on every page, `dark-mode.js`, link preview (dark) |
| Product icon | Page icon, favicon, touch icon, link preview, and any site that lists this one |
| Product colors | The tokens and their comments |
| Name, tagline, dark colors, hero image | Link preview |
| An image's size | Its `<img>` `width` and `height` |
| Root file added | `_config.yml` `exclude` |
| Brand mark, its backing, a shared component or script | Every sibling that has it |
| Files, conventions, behavior | `AGENTS.md`, `README.md` |

## New site

1. A site in a repo with other code publishes from an orphan `gh-pages` branch, edited in a worktree so `main`'s working tree is untouched: `git -C <repo> worktree add --orphan -b gh-pages ../<repo>-site`, or `git -C <repo> worktree add ../<repo>-site gh-pages` once it exists. A site that is its whole repo publishes from `main`.
2. Start from the closest sibling's base files, or write them per [`references/code.md`](references/code.md): the page, three sheets, `dark-mode.js`, `overview-return.js`, `_config.yml`, and `og-image/`. A product page has the hero, CTA row, and alternating feature sections; an interactive app keeps the hero's top bar and builds its own UI below it.
3. Pick the palette and a motif from the product, and build from the tokens and components in [`references/design-system.md`](references/design-system.md).
4. Make the icons, screenshots, and link preview per [`references/assets.md`](references/assets.md). Write copy per [`references/writing.md`](references/writing.md).
5. Add `CNAME` with the domain, and `ln -s AGENTS.md CLAUDE.md`.
6. To move a page from another site, copy it verbatim and leave the old path as a redirect: `location.replace()` plus a canonical link and a plain link to the new address. Keep both paths, since links elsewhere may never update.
7. If a hub site lists its siblings, add the new one: its card or link, any icons it copies, the hub's link preview and image alts, and the docs that list the siblings.

## Change an existing site

1. Read its `AGENTS.md`, and the product's own source when the change involves colors, icons, or screens.
2. Follow [`references/design-system.md`](references/design-system.md), [`references/code.md`](references/code.md), and the sibling that already does something similar.
3. Apply the "Update together" table, including siblings for shared patterns.

## Verify

Serve the root with `python3 -m http.server` rather than opening the file, so paths, modules, and workers behave as in production. Then check:

- Both themes from the OS setting and from the toggle, after a reload too, with `aria-pressed` following.
- With JavaScript off, the page still follows the OS theme.
- At 320px wide and at 200% text: no horizontal scroll, nothing clipped, nothing under the scroll-to-top button.
- Only the matching variant of each themed image downloads.
- `file images/og-image.png` reports 1200 x 630 and RGB rather than RGBA, and the PNG looks right when viewed.
- `git ls-files` at the root shows nothing unpublished that `_config.yml` doesn't exclude.

Headless Chrome follows the OS appearance (force light with `--blink-settings=preferredColorScheme=1`, dark with `--force-dark-mode`), and `--window-size` won't go narrower than 500px. For narrow widths, emulate the device over CDP (`Emulation.setDeviceMetricsOverride`), frame the page in a 320px `<iframe>`, or use a browser's responsive mode.
