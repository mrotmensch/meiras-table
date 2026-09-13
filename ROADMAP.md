# Meira's Table — content refactor + feature roadmap

## Context

Right now all 34 recipes live as one embedded JS array inside `index.html`, making the site impossible to edit without Claude. The immediate need is to split recipes into separate, human-editable files. While doing that, we're also scoping a small set of features the user wants built on top of this new foundation: bilingual search, a lightweight tagging system, a routed (not modal) recipe view for better stove-side readability, forward-compatible photo support, and link-preview metadata. This plan sequences all of it as phases that build on each other, to be executed incrementally.

No build tooling is available on this Mac (no Node/npm/npx), and the site is a static GitHub Pages deploy with a single `index.html` — every phase below is designed to work within that constraint (vanilla JS, no external libraries, no build step).

## Phase 1 — Split recipes into editable Markdown files

Replace the embedded `const recipes = [...]` array with:
- `recipes/<id>.md` — one file per recipe, frontmatter (`id`, `category`, `title_en`, `title_he`, optional `tags`, optional `image`) followed by `## Ingredients (EN)` / `(HE)` and `## Steps (EN)` / `(HE)` sections as plain Markdown lists, optional `## Notes (EN)` / `(HE)`. Confirmed format with the user already (example: spicy tomato salad, converted and approved).
- `recipes/manifest.json` — flat ordered array of recipe ids (drives display order; per-category grouping still comes from `categories` + `r.category`, unchanged).
- A hand-written parser in `index.html` (~30-40 lines, no dependency): split frontmatter on `---` delimiters into key/value pairs, then split the body on `## ` headings and turn each into a bilingual array (`ingredients.en/he`, `steps.en/he`) or string (`notes.en/he`).
- Data loading becomes async: fetch `manifest.json`, then fetch+parse each `recipes/<id>.md`, populate the existing `recipes` array, then call the router (see Phase 2) instead of `render()` directly.
- `categories` and `ui` strings stay inline in `index.html` — they're structural, not content that needs frequent editing.

This phase touches: `index.html` (remove embedded array, add loader/parser, keep `categories`/`ui`), plus 34 new `recipes/*.md` files + `recipes/manifest.json`.

**Local testing note:** `fetch()` requires the page be served over HTTP, not opened as `file://`. Use `python3 -m http.server` from the project root for local preview from here on (GitHub Pages already serves over HTTP, so production is unaffected).

## Phase 2 — Recipe detail becomes a routed page, not a modal

Currently clicking a card opens a fixed-position modal overlay (`.overlay`/`.sheet`). The user wants a dedicated page per recipe instead — easier to read at the stove — while keeping the sticky top nav (categories + language toggle) visible and usable.

- Cards become real links: `<a class="card" href="#recipe/${id}">` instead of `<button>` + click listener (also gets free "open in new tab" support).
- Add a hash router: `hashchange` listener + a `route()` function. If `location.hash` starts with `#recipe/`, render the single recipe into `main` (replacing the category grid) with a small "back to [category]" link at the top; otherwise render the existing homepage category grid as today. `route()` is called once after recipes finish loading (Phase 1), and on every `hashchange`.
- The lang-toggle click handler switches from calling `render()` directly to calling `route()`, so switching language while viewing a recipe re-renders that recipe page correctly instead of jumping back to the homepage.
- Set `document.title` per recipe (and reset it on the homepage route) — nicer browser tabs/bookmarks, and a building block for Phase 4's meta tags.
- Retire `.overlay` / `.sheet` / `.close-btn` CSS and the `openRecipe`/`closeRecipe` modal logic; the recipe-page render function replaces them, reusing the same ingredients/steps/notes layout markup that already exists in `openRecipe()` today.
- Add a `@media print` block scoped to the recipe page (hide nav, hero, footer, back-link; print just the recipe) — cheap now that a recipe is its own page rather than a modal.

Touches: `index.html` only (CSS + JS + card markup changes).

## Phase 3 — Bilingual search + tags

- **Tag vocabulary**: a small curated dictionary in `index.html`, e.g. `tagLabels = { vegan: {en:'Vegan', he:'טבעוני'}, 'dairy-free': {...}, quick: {...}, passover: {...} }`, extended as new tags are needed. Recipe frontmatter lists tag slugs from this vocabulary: `tags: vegan, passover`. Keeping a shared, curated dictionary (rather than fully freeform per-recipe tag text) avoids search fragmenting on typos/variants and keeps translation in one place instead of every recipe file.
- **Display**: no broad tag-chip UI — just a small, quiet annotation line at the bottom of the recipe page (e.g. "Tags: Vegan · Passover" / "תגיות: טבעוני · פסח"), matching the user's ask for something subtle rather than prominent.
- **Search box**: added to the sticky nav. Build a per-recipe search haystack once after load — `title.en + title.he + ingredients.en/he joined + tag labels (en+he)` — lowercased. Filter is substring match against this haystack, so a query works regardless of which script/language you type in or which language the UI is currently displaying (satisfies "works bilingually"), and a tag search like "vegan" surfaces every recipe carrying that tag even though the word doesn't appear in its title/ingredients. Filtering re-renders the homepage grid (category sections with no matches show nothing, or a "no matches" note reusing the existing `comingSoon`-style empty state).

Touches: `index.html` (nav markup for the search input, `tagLabels` dict, search/filter logic), plus a `tags:` line added to relevant `recipes/*.md` files as they're identified.

## Phase 4 — Shareable link previews (Open Graph)

Add `<meta property="og:title">`, `og:description`, `og:image` (the existing hero photo), `og:type`, `og:url`, and a plain `<meta name="description">` to `<head>`.

**Important limitation to flag:** these will be *site-wide*, not per-recipe. A crawler (WhatsApp, iMessage, etc.) reads meta tags without executing JavaScript, so a shared link to a specific recipe page would still show the generic site title/image, not that recipe's own title/photo. True per-recipe previews would need either a build step that pre-renders one static HTML file per recipe (not currently feasible without installing Node) or a server-side prerender proxy. Recommend shipping the site-wide version now; revisit per-recipe previews later if it turns out to matter in practice.

Touches: `index.html` `<head>` only.

## Phase 5 — Photo support (schema now, images added over time)

Design question the user asked directly: **single photo per recipe vs. a multi-photo step gallery** — my recommendation is to build for a single photo per recipe now, not a step gallery. Reasoning: a single hero photo is a small, additive schema field (`image: filename.jpg` in frontmatter) with a simple render rule (show it if present, fall back to the category icon if not) and no new UI pattern. A step-by-step gallery needs a fundamentally different data shape (an array of `{step, image, caption}`), a gallery/lightbox UI, and meaningfully more asset management — and most home recipes don't need it. It's also very likely photos get added recipe-by-recipe over months, which favors the simplest possible per-recipe unit.

To keep the door open without overbuilding: use `image: <filename>` (singular) as the baseline field now. If a specific recipe later genuinely wants a step gallery, that recipe can add an `images:` (plural) list instead — the render logic checks `images` first, falls back to `image`, falls back to the category icon — so this is opt-in per recipe rather than a structural rewrite later.

Concretely now: the optional `image` field is in the frontmatter schema and parser (Phase 1), and `recipes/images/<filename>` is referenced from the recipe detail page (larger photo, shown only when present). **Cards on the homepage grid stay icon-only regardless of `image`** — with only a handful of recipes photographed so far, a photo on some cards and icons on others looked inconsistent; the photo now only appears once you click into a recipe. Worth revisiting once most/all recipes have photos.

Touches: `index.html` (parser field, recipe-page render), `recipes/images/` (folder with its own README documenting the convention).

## Notes on things that need no structural change

- **Attribution/occasion notes** (e.g. "Savta's recipe, for Rosh Hashana") — the existing per-recipe `notes.en`/`notes.he` field, already rendered in italic at the bottom of the recipe view, already covers this. No schema change needed; just write that kind of note into the `## Notes` sections of the relevant recipe files.
- **Servings scaler** — considered and deliberately excluded. Ingredients are written as natural prose ("3-4 ripe, peeled red tomatoes"), not structured quantities; forcing a scalable format would mean rewriting every ingredient into a rigid qty/unit/item shape, undermining the handwritten-recipe voice. Not recommended.

## Verification

- After Phase 1: serve via `python3 -m http.server`, load the site, confirm all 34 recipes still render identically to before (spot-check a recipe with `notes` and one without), confirm both languages still work, confirm GitHub Pages deploy is unaffected (relative fetch paths).
- After Phase 2: click into a recipe, confirm nav/lang-toggle stay usable, confirm browser back button returns to the homepage at the right scroll position, confirm switching language while viewing a recipe re-renders that same recipe (not the homepage), test print preview on a recipe page.
- After Phase 3: search a Hebrew word while UI is in English (and vice versa) and confirm matches appear; search a tag name and confirm all recipes carrying that tag appear even without the word in their title/ingredients.
- After Phase 4: check rendered `<head>` manually (view source) for correct og:title/description/image; optionally paste the live URL into a link-preview debugger to confirm rendering.
- After Phase 5: add one real photo to one recipe, confirm it shows on both the card thumbnail and the recipe page, confirm every recipe without a photo still falls back cleanly to its category icon.
