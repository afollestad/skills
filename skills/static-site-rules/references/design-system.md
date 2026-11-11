# Design system

Class names and values are shared across sibling sites; reuse them before adding new ones.

## Identity

- **Palette.** Take the colors from the product: its icon, logo, or the colors its own UI uses. Each sibling picks its own hue family rather than a near neighbor of another's.
- **Both themes are designed.** The dark theme is the product's night, not an inverted light theme. Each theme sheet opens with a one-line comment saying what it depicts.
- **Motif.** Each site paints one backdrop from its product, such as a glow washing down from the top, a sky gradient, or a shape from its icon running behind the hero. The default is the `.hero-sky` glow below.
- **Glass** is for floating chrome only: ghost buttons, the scroll-to-top button, cards. A glass surface reads as clickable, so non-interactive groups are plain columns between hairlines instead.

## Tokens

Each theme sheet defines these on `:root`, after `color-scheme: light` or `dark`, and adds site-specific tokens below them.

| Token | Use | Light | Dark |
|---|---|---|---|
| `--bg` | Page canvas, on `html` | Near-white tinted with the hue | Near-black tinted with the hue |
| `--fg`, `--secondary` | Body and secondary text, at least 4.5:1 | Deep shades of the hue | Pale tints of the hue |
| `--accent` | Fills from the product's accent | The accent | The accent, lifted |
| `--on-accent` | Text on `--accent`, at least 4.5:1 | `#ffffff` | `--bg` |
| `--accent-text` | Small text, focus rings, hover, at least 4.5:1 | Accent deepened | Accent lifted |
| `--hairline` | Dividers, table rules, frame borders | `--fg` at .12 | `--fg` at .12 |
| `--surface` | Table and frame fill | `#ffffff` | `rgba(255, 255, 255, .04)` |
| `--chip-bg`, `--chip-fg` | Section labels | `--accent`, `--on-accent` | `--accent`, `--on-accent` |
| `--glass` | Glass fill | `rgba(255, 255, 255, .55)` | `rgba(255, 255, 255, .07)` |
| `--glass-border` | Glass edge | `--fg` at .1 | `rgba(255, 255, 255, .14)` |
| `--glass-shadow` | Glass lift | Hue's dark shade at .18 | `rgba(0, 0, 0, .35)` |
| `--icon-shadow` | Icon drop shadows | Hue's dark shade at .22 | `rgba(0, 0, 0, .45)` |
| `--media-shadow` | Screenshot frames | Hue's dark shade at .3 | `rgba(0, 0, 0, .55)` |
| `--logo-circle` | Shared brand mark's backing, 3:1 with the mark | Same on every sibling | Same on every sibling |
| `--title-gradient` | Hero name, every stop 3:1 on the glow | Deep accent shades, 100deg | Pale tint through the accent, 100deg |

`index-common.css` holds the theme-independent ones: `--font`, `--spring: cubic-bezier(0.34, 1.56, 0.64, 1)`, `--fab-size: 2.5rem`, and `--fab-inset: 1.5rem`.

## Type

| Element | Size | Weight | Tracking | Line height | Color |
|---|---|---|---|---|---|
| Body | `1rem`, `overflow-wrap: break-word` (splits only a word too long for a line of its own) | 400 | | 1.5 | `--fg` |
| Hero `h1` | `clamp(min(2.75rem, Nvw), 7vw, 4.25rem)`: N is the largest that fits the name on a 320px phone at 200% text and still gives `2.75rem` there at normal size (`20vw` for 7 or 8 letters, `13.75vw` for 10) | 700 | `-.035em` | 1.1 | `--title-gradient` |
| Tagline | `clamp(1.05rem, 1.6vw, 1.2rem)`, max `36rem` (widened just enough to set a longer one in two lines on desktop, plus about `2rem` for other systems' fonts), `text-wrap: balance` | 400 | | 1.55 | `--secondary` |
| Section `h2` | `clamp(1.9rem, 3.5vw, 2.6rem)`, `1rem` below its chip; floor `min(1.9rem, 10vw)` when a heading holds a word too long for a 320px phone at 200% text | 700 | `-.03em` | 1.1 | `--fg` |
| Feature copy | `1.1rem`, `1rem` below the `h2` | 400 | | 1.6 | `--secondary` |
| Chip | `.875rem` | 600 | | | `--chip-fg` |
| Button | `.95rem` | 600 | | | |
| Footer | `.875rem` | 400 | | | `--secondary` |

- Font stack in `--font`: `-apple-system, BlinkMacSystemFont, "SF Pro Text", "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif`, with `-webkit-font-smoothing: antialiased`.
- The `h1` gradient applies inside `@supports ((-webkit-background-clip: text) or (background-clip: text))`, with `-webkit-text-fill-color: transparent`, over a plain `--fg` fallback.
- `text-wrap: pretty` on paragraphs that sit side by side.

## Layout

- `.container`: `max-width: 68rem`, `margin: 0 auto`, `padding: 0 1.25rem`.
- `.section-hero`: `padding: 6rem 0 5rem`, centered text, `position: relative`, `overflow: hidden` then `overflow: clip`, so the glow never scrolls the page sideways.
- `.section-feature`: `padding: 5rem 0`.
- `.feature-split`: `max-width: 60rem`, two `1fr` columns, `gap: 4rem`, vertically centered. `.feature-text` is `max-width: 28rem`, centered in its column. `.feature-split-reverse` puts the text second; alternate it section by section.
- `.section-centered`: centered text, its copy at `max-width: 36rem`, for full-width media.
- `.hero-media`: `max-width: 56rem`, `margin: 4rem auto 0`.

## Components

- **Glow** (`.hero-sky`): absolute, `inset: 0`, `pointer-events: none`, masked `linear-gradient(to bottom, #000 60%, transparent 100%)` so it fades before the hero's clipped edge. The theme sheet paints a wash of the hue in an `ellipse 1100px 520px at 50% 0%` (`560px` in dark) and a second color in an `ellipse 700px 420px at 82% 18%`. Hero content above it gets `position: relative; z-index: 1`.
- **Top bar.** `.top-brand` (left) and `.top-actions` (right) are absolute at `top: 1rem`, `1.5rem` from the sides, `2.5rem` tall flex rows, actions `gap: .5rem`.
- **Brand mark.** `.brand-logo` link: `padding: .25rem`, `border-radius: 50%`, `--logo-circle` fill, `box-shadow: 0 2px 6px var(--icon-shadow)`, the mark `2rem` wide and uncropped.
- **Top buttons.** `.top-btn`: `2.5rem` circles, no border or fill, `opacity: .75`, rising to 1 on focus and on hover, where they also take `--accent-text`, `transition: opacity .2s ease, color .2s ease`. The theme toggle comes last, with 20px sun and moon icons hidden by default; the light sheet shows `.icon-moon` and the dark sheet `.icon-sun`.
- **Product icon.** `.app-icon`: `7rem` wide, `filter: drop-shadow(0 .6rem 1.2rem var(--icon-shadow))`, the `h1` `1.5rem` below it.
- **CTA row.** `.cta-row`: centered wrapping flex, `gap: .75rem`, `margin-top: 2rem`.
- **Buttons.** `.btn-primary` and `.btn-ghost`: `min-height: 3rem`, `max-width: 100%`, `padding: .4rem 1.3rem` (more pushes a bordered two-line button past `3rem`), `border-radius: .55rem`, `gap: .5rem` around an 18px icon with `flex-shrink: 0`, no underline, `transition: transform .3s var(--spring)`, `scale(1.03)` on hover. Labels may wrap, so on a phone with enlarged text a button grows taller rather than wider than the page; never `white-space: nowrap`. Let each button size to its label; never widen it with extra padding or a `min-width` to match its neighbor, which insets the content. Primary is `--accent` with `--on-accent` text and `box-shadow: 0 .5rem 1.25rem -.5rem var(--accent)`; ghost is glass with `--fg` text.
    - **Two-line labels.** A button's label is a `.btn-stack` (column, left-aligned with `text-align: left` for when its lines wrap, `line-height: 1.15`) ending in a `.btn-subtext` line (`.7rem`, 500), which keeps it at `3rem`. A direct download's subtext states its requirements tersely ("macOS 26+", or "macOS 26+, Silicon" for an arm64-only build) in full `--on-accent`, since a muted color fails 4.5:1 on the accent; GitHub's reads "Stars appreciated!" in `--secondary`.
    - **App Store.** An App Store link is a primary button, not Apple's badge: Apple's logo from the badge artwork in `fill="currentColor"`, and a stack leading with its subtext, "Download on the" over "App Store", so it reads in order.
- **Glass.** `--glass` fill, `1px solid var(--glass-border)`, `backdrop-filter: blur(12px) saturate(160%)` with its `-webkit-` copy.
- **Chips.** `.chip`: `inline-flex`, `padding: .3rem .85rem`, `border-radius: 999px`, `--chip-bg` fill, `box-shadow: inset 0 0 0 1px rgba(255, 255, 255, .2), inset 0 1px 0 rgba(255, 255, 255, .3), 0 1px 2px rgba(0, 0, 0, .1), 0 .35rem .9rem -.35rem var(--chip-bg)`. On a page with section headlines the chip is a `<p>` above the `h2`; otherwise the chip can be the `h2`.
- **Media frames.** `.media-frame`: `border-radius: 1rem`, `1px solid var(--hairline)`, `--surface` fill, `overflow: hidden`, `box-shadow: 0 2px 4px rgba(0, 0, 0, .08), 0 1.5rem 3rem -1rem var(--media-shadow), 0 3rem 6rem -2rem var(--media-shadow)`. Screenshots inside are square-cornered and `width: 100%`.
- **Cards.** Glass, `border-radius: 1.25rem`, `padding: 1.75rem 1.25rem`, `box-shadow: 0 .5rem 1.5rem -.75rem var(--glass-shadow)`, lifting `translateY(-4px)` with a deeper shadow on hover and on `:focus-visible`, `transition: box-shadow .2s ease, transform .3s var(--spring)`. A whole card is one link. Choose column counts so rows stay full, and keep side-by-side descriptions about equal in length.
- **Footer.** `.site-footer`: centered, `padding: 4rem 1.25rem calc(var(--fab-inset) + var(--fab-size) + 1rem)` so its text clears the scroll-to-top button. "Built by <author> • Open source (<license>)", `&#8226;` separators, links without underlines until hovered (then `--accent-text`), `aria-current="page"` on the current page's link. Data sources go on a second line.
- **Scroll-to-top.** `.overview-return`: fixed `--fab-inset` from the bottom right, `--fab-size` glass circle, `box-shadow: 0 4px 16px var(--glass-shadow)`, `z-index: 300`. Hidden (`opacity: 0`, `visibility: hidden`, `translateY(.5rem)`) until the page scrolls past 400px, when `.visible` shows it; `scale(1.05)` on hover; `transition: opacity .25s ease, transform .3s var(--spring), visibility .25s ease`.
- **Tables.** Hairline rules, `--surface` fill, uppercase `.85rem` headers in `--secondary`. Tables of three or more columns stack into one card per row on phones, labeling each cell from its `data-label`.
- **Focus.** `a:focus-visible, button:focus-visible`, plus any frame focusable because it scrolls: `outline: 2px solid var(--accent-text)`, `outline-offset: 2px`.

## Motion

- Transforms spring with `--spring` over `.3s`; opacity and color ease over `.2s`, fades over `.25s`.
- Buttons scale to 1.03 on hover, the scroll-to-top button to 1.05, and cards lift 4px.
- The last block of the sheet is `@media (prefers-reduced-motion: reduce)`: transitions and animations to `.01ms !important`, hover transforms to `none`, and `scroll-behavior: auto` where `html` scrolls smoothly.

## Responsive

- At 820px: feature splits stack into one centered column with `gap: 2.5rem`, text `max-width: 34rem` and first, and sections drop to `padding: 4rem 0`.
- At 600px: the hero drops to `padding: 5rem 0 3.5rem`, the icon to `5.5rem`, `.hero-media` to `margin-top: 3rem`, and media frames to `border-radius: .75rem`.
- Pixel media queries miss enlarged text, so catch columns that could overflow with an `em` media query or a container query.
- Support 320px wide and 200% text with no horizontal scroll. Measure text nodes too: a long word runs out of its box without widening it, and a hero's `overflow: clip` hides it instead of scrolling.
- Content that can't wrap without losing meaning, like a file tree, scrolls inside its own frame (`overflow-x: auto`, `white-space: nowrap`) with `tabindex="0"` so the keyboard can reach it.
