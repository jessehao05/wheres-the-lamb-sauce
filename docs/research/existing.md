# Existing projects

Findings from 2026-10-04: a search of GitHub and the web for projects that scrape CBORD NetNutrition.

**Main finding:** [Kratosxx24/anderson-dining](https://github.com/Kratosxx24/anderson-dining) already scrapes Vanderbilt's NetNutrition. It collects ingredients and nutrients and shows them on a filterable page. It's a Python project, created 2026-09-02 and still active (last push 2026-10-04), with 0 stars and no license.

## Projects found

| Repo | School | Relevance |
|---|---|---|
| [Kratosxx24/anderson-dining](https://github.com/Kratosxx24/anderson-dining) | Vanderbilt | **Very high.** Uses `requests` + BeautifulSoup and runs every 3 hours on GitHub Actions. It saves every label (macros, micronutrients, full ingredients, allergens) to JSON and serves a static page on GitHub Pages. |
| [Naimy441/Naimy441.github.io](https://github.com/Naimy441/Naimy441.github.io) | Duke | High. Direct HTTP scraper (its older Selenium version is retired) that collects menus, labels and ingredients. Very active (about 1,250 commits). |
| [NCSU-App-Development-Club/FoodFinder](https://github.com/NCSU-App-Development-Club/FoodFinder) | NC State | Medium. A Kotlin/Ktor backend with a daily NetNutrition scraper, SQLite storage and limits on concurrent requests. Its README doesn't say if it collects ingredients. |
| [ndg24/vandy-menu](https://github.com/ndg24/vandy-menu) | Vanderbilt | Low. Next.js PWA whose data comes from Claude Code driving a browser once a week. Menu items only, no nutrition. |
| [jpes707/netnutrition-app](https://github.com/jpes707/netnutrition-app) | Duke | Low. iOS class project from 2022, now abandoned. |

[mattorib/cbord-api](https://github.com/mattorib/cbord-api) came up in the search but isn't relevant: it targets CBORD GET, the card/payments system, not NetNutrition. No npm or PyPI package that wraps NetNutrition was found.

## What anderson-dining's code answers

These points were checked in its source files ([scraper/](https://github.com/Kratosxx24/anderson-dining/tree/main/scraper)). They answer several open questions in [netnutrition.md](netnutrition.md).

- **`X-Requested-With: XMLHttpRequest` is required.** Without it, the endpoints return a full HTML page instead of the JSON envelope (`fetcher.py`). They also send `Referer: <base>/`.
- **`detailOid` is not stable.** Their README says it identifies one serving on one menu, not the recipe: "Spinach" had 16 different OIDs in a single day, and each new day mints new ones. (A comment in `fetcher.py` claims the opposite, but the README backs its claim with measurements.) Fetching different OIDs for the same item returns byte-identical labels, so they key the cache by `name|serving` (lowercased, whitespace collapsed).
- **The label call also takes `menuOid`.** Their request is `POST /NutritionDetail/ShowItemNutritionLabel` with `{detailOid, menuOid}`. A code comment says any `menuOid` that contains the item works.
- **There's an hours endpoint.** `POST /Unit/GetHoursOfOperationMarkup` with `{unitOid}` returns the weekly hours (`hours.py`), and their page works out open/closed in the browser. That covers the "currently" search mode.
- **Incomplete labels can be detected.** Items with missing nutrient data carry the CSS class `cbo_nn_LabelPrimaryDetailIncomplete` (`nutrition.py`), so those values should be treated as minimums.
- **Request volume is lower than estimated.** They report about 1,300 menu rows a day collapsing to about 500 unique labels. They cap new label fetches at 400 per run (`NUTRITION_BUDGET`) with a 0.6 s delay between requests (`REQUEST_DELAY`), and scrape today plus tomorrow (`DAYS_AHEAD = 2`).

## Things to weigh

- **Overlap.** Their page already filters by day, meal, open-now, traits, allergens and free text, and sorts by protein or calories. It doesn't search ingredients, filter by nutrient, or search for several ingredients and nutrients at once, so that's where this project would stand out. See [Adapting them to this project](#adapting-them-to-this-project).
- **No license.** Their code can be read to learn the protocol, but it can't be copied. Ask the author before reusing or contributing.
- **Not a model for etiquette.** They send a fake Chrome user agent and nothing in the code mentions robots.txt. That doesn't settle the robots.txt issue in [netnutrition.md](netnutrition.md).
- **Possible collaborator.** The author is likely another Vanderbilt student working on the same problem.

## Adapting them to this project

Both repos were cloned and read on 2026-10-04. The features being compared are the ones in [overview.md](overview.md).

**Summary:** vandy-menu would need almost everything rebuilt, because its data has no nutrition, no ingredients and no individual items. anderson-dining already has all the data these features need, so getting them is mostly front-end work in one file.

### vandy-menu

**What it is:** a Next.js 16 + React 19 + TypeScript + Tailwind 4 PWA on Vercel. All its data is in `data/menu.json`, which someone regenerates by hand each week by having Claude Code drive a browser. The JSON is bundled into the site at build time.

**Why the data won't work:**
- **It's a summary, not the menu.** The whole week across 8 halls is just 247 entries. Rand's Monday lunch is 5 lines like `"GRILL": "Beyond Burger Patty & Blended Beef Burger Patty & Fried Chicken Tenders"`; NetNutrition has about 223 items for that one meal.
- **No nutrition, ingredients, serving sizes or item IDs.** Each entry is `{station, dish}`.
- **Only 8 of the 21 locations.**
- **Hours are hand-entered** (`data/hours.json`, dated 2026-09-01), and the data refresh is manual and weekly.

**What could be kept:** the UI shell, meaning the day/hall/meal selectors, the PWA setup and `src/lib/hours.ts` (a working "open now" check in Chicago time). Its `currentMealPeriod()` in `src/lib/menu.ts` uses the browser's time zone rather than Chicago's, a minor bug.

**What would have to be built:**
1. A real scraper, i.e. what [netnutrition.md](netnutrition.md) describes or what anderson-dining already does.
2. A new item-level data model with nutrients and ingredients in `src/lib/types.ts`.
3. A scheduled job (GitHub Action or Vercel cron) to replace the manual refresh.
4. A new cross-hall search view, since the app is built around picking one hall and then viewing its menu.
5. Probably loading the data at runtime instead of at build time, because full nutrition data is around 1.5 MB.

Steps 1–3 are the hard part of this project, so starting from vandy-menu saves very little.

### anderson-dining

**What's already there:** its data from 2026-10-04 covers 14 locations and 1,118 menu rows. Of its 1,267 cached recipes, 1,262 have nutrients and 1,232 have ingredient lists. Hours and open/closed status are included. The front-end is one 1,638-line `docs/index.html` with no build step.

| Feature | What exists | What to change |
|---|---|---|
| **Whole day vs. currently** | Mostly there: date and meal chips plus an "Open now" chip | "Open now" hides closed halls, but an open hall still shows its other meals' menus. A "currently" mode should also select the current meal. Small change. |
| **Search by ingredient** | Missing. `itemPasses()` only matches item names | Also match `macrosFor(item).ingredients`. A couple of lines, plus word-boundary matching. "beef" matches 8 item names on 2026-10-04 but appears in 74 recipes' ingredients, which is exactly the gap this project targets. |
| **Search by nutrient + show amount** | Partial. Sorting only covers protein and calories; other nutrients appear only when an item is expanded | Add a nutrient picker with an optional minimum, generalize `sortItems()` to any field in `NUTRIENT_FIELDS`, and show the matched nutrient's amount on each row. Values in the `incomplete` list should show as minimums. |
| **Several ingredients/nutrients at once** | Missing, and it's the biggest change | The overview's example is an AND across a hall, not within one dish: a hall passes if it has *some* dish with zinc and *some* dish with vitamin C. `render()` filters dish by dish, so this needs per-term matching for each hall, keeping halls where every term has a hit and labeling which dish matched which term. |
| **Results grouped by hall** | Already done | — |
| **Daily data pull** | Already done (every 3 hours, today + tomorrow) | — |

The existing view is built for browsing menus, while this project's flow starts with a search. So rather than adding more filter chips, a separate "Search" view could sit next to their three existing views (Menus, Taste of Nashville, Tracker). It can reuse their helpers (`macrosFor`, `hallOpenState`, `hallServesMeal`). Rough estimate: a few hundred lines of JavaScript.

**Limits of their data that would carry over:**
- **Nutrients:** calcium, iron, potassium and vitamins C and D. No zinc and no vitamin A, because vitamin A only appears in the bulk grid.
- **Ingredient text is noisy.** "beef" also matches things like beef base in a sauce, so what counts as a match needs deciding.
- **Missing locations:** places with no published menu (Wasabi, the Suzie's locations, the food truck) don't appear.

### Three ways to use anderson-dining

1. **Contribute upstream.** Send the search view as a PR. This is the least work, and it means working with someone already maintaining the scraper.
2. **Fork it.** The repo has no license, so this needs the author's permission.
3. **Build a separate front-end on their published data.** Their files are public at `https://kratosxx24.github.io/anderson-dining/menu.json` (~309 KB) and `https://kratosxx24.github.io/anderson-dining/nutrition.json` (~1.5 MB); both loaded fine on 2026-10-04. This needs no scraper, and this project wouldn't contact NetNutrition at all, but the app would depend on their uptime and file format. Ask them first in this case too.

Whichever option is chosen, contact the author first. Options 1 and 3 both skip the scraper entirely.
