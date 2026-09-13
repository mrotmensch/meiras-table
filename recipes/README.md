# Adding & editing recipes

Each recipe lives as its own file in this folder: `recipes/<id>.md`. No coding tools needed — any plain text editor works.

## Format

```markdown
---
id: my-recipe-id
category: salads-soups
title_en: My Recipe
title_he: המתכון שלי
tags: vegan, quick
image: my-recipe-id.jpg
---

## Ingredients (EN)
- item one
- item two

## Ingredients (HE)
- פריט אחד
- פריט שני

## Steps (EN)
1. Do this.
2. Then this.

## Steps (HE)
1. עושים ככה.
2. ואז ככה.

## Notes (EN)
Optional note — good for provenance ("Savta's recipe, for Rosh Hashana") or clarifications.

## Notes (HE)
הערה אופציונלית.
```

- `id` must match the filename (without `.md`) and be unique.
- `category` must be one of: `salads-soups`, `fish-meat`, `sides`, `breads`, `shabbat-morning`, `sweet`.
- `tags` and `image` are optional — omit the line entirely if not used.
- `tags` is a comma-separated list of existing tag slugs (see below). Tags show up as a small line under the recipe and are searchable.
- `image` is just a filename inside `recipes/images/` — see that folder's README.

## Adding a new recipe

1. Create `recipes/<id>.md` following the format above.
2. Add `"<id>"` to `recipes/manifest.json`, in the position you want it to appear (recipes are grouped by category automatically; order within a category follows manifest order).
3. Commit and push — the live site rebuilds automatically within a minute or two.

## Removing a recipe

Delete its `.md` file and remove its id from `manifest.json`.

## Tag vocabulary

Tags are a small curated set defined in `index.html` (search for `tagLabels`), currently: `vegan`, `dairy-free`, `quick`, `passover`. To add a new tag, add it to that dictionary (with an English and Hebrew label) — then any recipe file can use it.
