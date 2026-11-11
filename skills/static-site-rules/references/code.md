# Code

## Files

```
index.html               The home page; other pages sit beside it
styles/index-common.css  Everything theme-independent
styles/index-light.css   Light tokens and theme-painted backgrounds
styles/index-dark.css    Dark tokens and theme-painted backgrounds
styles/<page>.css        A secondary page's own layout, if it needs one
scripts/dark-mode.js     Theme override
scripts/overview-return.js  Scroll-to-top button
scripts/platform-gate.js  Download gate, on a page with a direct Mac download
images/                  Icons, screenshots, link preview
og-image/                Link preview source and renderer, unpublished
favicon.png, apple-touch-icon.png, CNAME, _config.yml
AGENTS.md, CLAUDE.md     Agent docs (CLAUDE.md symlinks to AGENTS.md), unpublished
```

## Theming

- `<head>` links `index-common.css`, then any page sheet, then `<link rel="stylesheet" href="styles/index-dark.css" id="theme-dark" media="(prefers-color-scheme: dark)">` and `<link rel="stylesheet" href="styles/index-light.css" id="theme-light" media="(prefers-color-scheme: no-preference), (prefers-color-scheme: light)">`, then `<script src="scripts/dark-mode.js"></script>` as a classic, synchronous script.
- Two `<meta name="theme-color">` tags, `media="(prefers-color-scheme: light)"` and `"(prefers-color-scheme: dark)"`, each with its theme's `--bg`.
- Each theme sheet sets `html { background: var(--bg); }` and shows its toggle icon.
- A themed image is a `<picture>`: `<source srcset="…-dark.webp" type="image/webp" data-theme="dark" media="(prefers-color-scheme: dark)">`, then the light `<img>`. Only the matching one downloads.
- A gradient using `in oklab` follows an sRGB copy with literal colors, since a `var()` in the second declaration would defeat the fallback.

`dark-mode.js` must:

1. Read `localStorage.theme` inside `try`, since storage throws in private modes and an uncaught throw in `<head>` leaves the page unthemed; write it inside `try` too.
2. Do nothing more without a stored `light` or `dark`, since the media-scoped sheets, sources, and metas already follow the OS, including live changes.
3. With one, set the chosen sheet's `media` to `all` and the other's to `not all`, set every `theme-color` meta's `content` to that theme's `--bg`, and set each `picture source[data-theme]`'s `media` to `all` or `not all` by its theme, again on `DOMContentLoaded`, since no `<picture>` exists yet in `<head>`.
4. On `DOMContentLoaded`, wire `#theme-toggle`: `aria-pressed` is `true` in dark, and a click flips the theme, stores it, and applies it.
5. While no override is stored, follow `matchMedia('(prefers-color-scheme: dark)')` changes to keep `aria-pressed` and the next click's direction right.

## Scroll-to-top

- `<button class="overview-return" id="overview-return" type="button" aria-label="Scroll to top">` with an up-arrow icon, before `</body>`, with an HTML comment saying not to rename it. Ad blockers' cosmetic filters hide `back-to-top`, `jump-to-top`, and `scroll-to-top`.
- `overview-return.js` toggles `.visible` past `scrollY` 400 from a passive scroll listener and on load, and its click calls `scrollTo({ top: 0 })` with `behavior: 'smooth'`, or `'auto'` under `prefers-reduced-motion: reduce`.

## Download gate

- A direct Mac download is `#release-link`, an id as non-generic as the scroll-to-top button's, and ships both subtexts: `.download-label` and `.download-label-unavailable` ("Only for Macs").
- `platform-gate.js`, copied verbatim from a sibling, runs as a classic, synchronous script in `<head>` so the inert state paints on the first frame: off a Mac it adds `not-macos` to `<html>` and makes `#release-link` an inert, `aria-disabled` link.
- Under `.not-macos`, the unavailable subtext replaces the other and the button fades: `opacity: .45`, no shadow, `cursor: default`, `pointer-events: none`.

## CSS

- Four-space indent, one declaration per line, decimals without a leading zero (`.5rem`, `.2s`, `rgba(0, 0, 0, .1)`) except inside `cubic-bezier()`, `rem` for sizes.
- Sections are banner comments 78 characters wide, in page order: `/* Hero ------...--- */`, ending with `Responsive`, whose last block is `prefers-reduced-motion`.
- Each sheet opens with a comment on what it holds. Comment the why above any non-obvious rule, and name where a value comes from and what must change with it.
- Hover styles only inside `@media (hover: hover) and (pointer: fine)`.
- Write the fallback first: `overflow: hidden` then `overflow: clip`; `100vh` then `100svh`; `cqi` inside `@supports (width: 1cqi)`. Prefix `-webkit-backdrop-filter`, `-webkit-mask-image`, `-webkit-background-clip`, and `-webkit-text-size-adjust`.
- Base: `*, *::before, *::after { box-sizing: border-box; }`, `text-size-adjust: 100%`, `margin: 0` on headings and paragraphs, `img { display: block; max-width: 100%; height: auto; }`, and `picture { display: block; }` so an image doesn't sit in an inline box with a gap under it.
- Scope page-specific type to its section (`.hero-copy h1`, `.section-feature h2`), so it doesn't reach other pages.
- Use `env(safe-area-inset-*)` with `viewport-fit=cover` only on full-bleed app UI.

## HTML

- `<!DOCTYPE html>`, `<html lang="en">`, four-space indent, wrapped attributes continued eight spaces in, a blank line between `<head>` groups and between top-level sections.
- `<head>` order: `charset`, `width=device-width, initial-scale=1` viewport (never `user-scalable=no`), title, description, canonical on secondary pages, the `theme-color` pair, Open Graph (`og:title`, `og:type`, `og:url`, `og:description`, `og:image` with `:type`, `:width`, `:height`, and `:alt`), Twitter (`summary_large_image`, `twitter:image`, `twitter:image:alt`, since Twitter never reads `og:image:alt`), icons (`favicon.png` at `sizes="96x96"`, `apple-touch-icon.png`), stylesheets, scripts.
- Every published `<img>` has `width`, `height`, and `decoding="async"`, plus `fetchpriority="high"` for the hero or `loading="lazy"` below the fold. Decorative ones get `alt=""`.
- Icons are inline Feather-style SVGs: `viewBox="0 0 24 24"`, `fill="none"`, `stroke="currentColor"`, `stroke-width="2"`, round caps and joins, `aria-hidden="true"`, 20px in the top bar.
- Icon-only controls have an `aria-label`; a toggle's label names its on state ("Dark mode") and `aria-pressed` carries the state.
- Buttons have `type="button"`. Links open in the same tab.
- Comment non-obvious markup in HTML comments, as the CSS does.

## JavaScript

- Page scripts are classic scripts in an IIFE with `var` and `function`: no modules, no globals.
- Interactive app logic is ES modules: one responsibility per module, browser-free logic in pure modules tested with `node --test`, and UI modules that take elements and callbacks.
- Feature-detect rather than sniff user agents, and give a missing API a tested fallback.
- Vendored code lives unmodified in `vendor/<name>/`, with its version in `AGENTS.md` and its license published.

## Publishing

- GitHub Pages runs Jekyll on the branch root and copies files without front matter verbatim, so add none. It drops names starting with `_` or `.`.
- `_config.yml` lists every root file that isn't part of the site in `exclude` by name, since `**/*.md` misses the root:

  ```yaml
  # Names files outright, since Jekyll's "**/*.md" doesn't match the repo root.
  exclude:
    - og-image/
    - AGENTS.md
    - CLAUDE.md
    - README.md
  ```

- Publish third-party licenses that require it as `.txt`, so Markdown exclusions can't drop them.
- `CNAME` holds the custom domain.
