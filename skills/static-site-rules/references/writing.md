# Writing

## Page copy

- Plain and concrete: say what the product does, in the reader's terms, with no hype. Second person in feature copy. Sentence case for headings.
- **Tagline:** one or two sentences naming what it is, then what you do with it. "Turn recipes into a shopping list. Pick the servings, and check items off aisle by aisle."
- **Meta description:** "<Name> is <what it is>: <what you do with it>." `og:description` is the same without "<Name> is".
- **Chip:** one or two words naming the feature ("Servings", "Lists"). **Section `h2`:** a short headline about the benefit ("Shop once for the whole week"). **Feature copy:** one to three sentences.
- **Cards side by side:** descriptions of about equal length, so they wrap to the same line count.
- **Alt text:** describe what's on screen, including its visible text and values, as one sentence a listener can picture. "The shopping list for two recipes, grouped by aisle, with butter and eggs checked off." Decorative images, including an icon beside the name, get `alt=""`.

## Comments and `AGENTS.md`

- Say why, in full sentences, in the present tense: the constraint, the browser behavior, or the coupling. "Both theme-color metas are media-scoped, so an override has to overwrite each of them."
- Name the source of truth and what must change with it: "The icon's accent, 4.8:1 on white." "Don't forget to update the theme colors in scripts/dark-mode.js if --bg changes."
- Warn in caps only to stop a plausible mistake ("Do NOT rename this to a generic 'back to top' name").
- `AGENTS.md` opens with the domain as its title and a paragraph on how the branch publishes, then `Look`, `Theming`, and `Images` (`Screenshots`, `Icons`, `Link preview`). It records every regeneration recipe and coupling, and changes in the same commit as the code it describes. `CLAUDE.md` is a symlink to it.
