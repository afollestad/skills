# Assets

Convert non-sRGB images to sRGB first, since browsers read untagged pixels as sRGB and wide-gamut colors come out dull.

## Icons

- **Product icon.** A square image with its plate and corner mask baked in, at 256x256, as `images/app-icon-light.png` and `-dark.png` when it has a dark variant. Show it as a `<picture>` with the dark `<source>`, and shadow it with `filter: drop-shadow(...)`, which follows its alpha.
- **Favicon.** `favicon.png`, the light icon at 96x96, in both themes: Safari draws its own backdrop behind a favicon too dark for its tab bar and reads icon links once at load, so a themed one could neither look right nor follow the toggle. Add `favicon.ico` (16, 32, 48) only for old browsers.
- **Touch icon.** `apple-touch-icon.png`, the light icon at 180x180, flattened onto an opaque color from its plate, since the home screen masks it and fills transparent corners with black.
- **Copies on other sites.** A hub that shows sibling sites' icons copies them under a stable name and re-copies them when they change. A shared brand mark is never copied (see `SKILL.md`).

## Screenshots

- Capture each screen in both themes from the same fixed demo data, so the sets match and one `alt` describes both.
- Downscale captures to the size they display at, at most 2x.
- Encode UI captures as lossless WebP, since lossy ringing shows around text: `cwebp -lossless -z 9 in.png -o images/<screen>-light.webp`.
- Name them `<screen>-<light|dark>.webp`, with a prefix when one screen comes in several sizes. An image that doesn't change with the theme ships once.
- Ship each as a `<picture>`: dark `<source type="image/webp" data-theme="dark" media="(prefers-color-scheme: dark)">`, light `<img>` fallback, with `width` and `height` set to the file's size.
- Record how each image is made in `AGENTS.md`, so it can be regenerated.

## Link preview

- `images/og-image.png`: 1200x630, opaque, PNG (crawler support for WebP is patchy), on the dark theme, since a preview can't know the reader's theme and composites transparency unpredictably.
- Layout: the icon, the name in the title gradient, and a one-line tagline on the left; the hero image on the right, cut off by the frame's edges; the site's motif behind.
- `og-image/og-image.html` is a 1200x630 page that links `../styles/index-common.css` and `../styles/index-dark.css` without `media`, so the preview can't drift from the page. It reuses the hero's classes, with overrides for the fixed size and a left-aligned layout.
- `og-image/render.sh` screenshots it, finding Chrome or taking `CHROME_PATH`, with a throwaway profile (`profile=$(mktemp -d)`, removed by `trap 'rm -rf "$profile"' EXIT`) so it never touches a running browser: `--headless=new --force-dark-mode --force-device-scale-factor=1 --hide-scrollbars --user-data-dir="$profile" --window-size=1200,630 --virtual-time-budget=5000 --screenshot="$PWD/images/og-image.png" "file://$PWD/og-image/og-image.html"`. It deletes the old PNG first, so a failed render shows up as a missing file rather than a stale preview.
- `_config.yml` keeps `og-image/` unpublished. After rendering, check `file images/og-image.png` and view the PNG for overflow.

## Badges

Keep third-party badges and marks their owner's artwork, unaltered and at least their minimum size.
