# NetNutrition research

Source: https://netnutrition.cbord.com/nn-prod/vucampusdining (CBORD NetNutrition, Vanderbilt Campus Dining)

Findings from 2026-10-04, when the site was tested directly with curl.

## How the site works

NetNutrition is an ASP.NET app that keeps each visitor's state in a server-side session. The page's JavaScript (`/nn-prod/Scripts/cbord_nn_ui_repsonsive.js`) makes plain POST requests to `/nn-prod/vucampusdining/<Controller>/<Action>`. These return JSON containing HTML fragments, or plain HTML. Logging in isn't required, and there's no bot protection.

Base URL: `https://netnutrition.cbord.com/nn-prod/vucampusdining`

| Step | Request | Returns |
|---|---|---|
| 0 | `GET /` (follow the redirect) | Sets the `ASP.NET_SessionId` cookie. The home page lists every location as `unitsSelectUnit(N)` with an Open/Closed badge. |
| 1 | `POST /Unit/SelectUnitFromUnitsList` with `unitOid=<N>` | JSON `{success, panels:[{id, html}]}`. The `menuPanel` HTML lists that location's menus by date, e.g. `menuListSelectMenu(9492390)` → "Lunch" under "Monday, October 5, 2026". About 6 days are posted ahead. |
| 2 | `POST /Menu/SelectMenu` with `menuOid=<id>` | JSON. The `itemPanel` HTML is a table of items grouped by station (`data-categoryid`), with serving size and allergen/trait icons (`<img title='Dairy'>`, etc.). Each item has an ID in `getItemNutritionLabelOnClick(event,<detailOid>)`. |
| 3 | `POST /NutritionDetail/ShowItemNutritionLabel` with `detailOid=<id>` | HTML nutrition label: serving size, calories, fat, sat/trans fat, cholesterol, sodium, carbs, fiber, sugars, protein, Vit D, calcium, iron, potassium, Vit C, and **the full ingredient list**. |
| 3b | `POST /Menu/SelectItem` with `detailOid=<id>` for each item, then `POST /NutritionDetail/ShowMenuDetailNutritionGrid` (empty body) | One HTML table with macros for every selected item, plus Vit A/C, calcium, iron, Vit D (%DV). **No ingredients.** |

Step 2 returns **every dish on the menu**. Most rows have `display:none` because their stations start collapsed in the UI, but they're all in the response, so nothing needs to be expanded. The Rand Monday lunch response had 223 distinct items.

Other endpoints seen in the JS but not tested:
- `Home/HandleNavBarSelection`: takes `unit`, `meal`, `date` and `typeChange`. It drives the "Show All Units" and "Show All Dates" dropdowns.
- `Menu/SelectTrait`: server-side allergen filter. It's saved in the session and hides matching dishes, so the scraper should never call it.
- `openHoursInModal(unit)`: location hours. The data source wasn't checked.

### Gotchas

- Because state is server-side, each session must make its calls in order: select a unit, then a menu, then an item. Use one session per worker and don't run calls in parallel within a session.
- A POST with no parameters still needs a body (`-d ""`). Otherwise the server returns HTTP 411 Length Required.
- The first `GET` returns a 302 redirect that sets the cookie, so follow redirects (`curl -L`) and keep a cookie jar.
- The `X-Requested-With: XMLHttpRequest` header was sent on all POSTs. Whether it's required wasn't tested.

### Unit IDs

| ID | Location | ID | Location |
|---|---|---|---|
| 1 | Rand Dining Center | 12 | Commons Munchie |
| 2 | The Commons Dining Center | 13 | Kissam Munchie |
| 3 | The Kitchen at Kissam | 14 | Highland Munchie |
| 4 | The Pub at Overcup Oak | 15 | Highland Hotline |
| 5 | VandyBlenz | 16 | Local Java Cafe at Alumni |
| 6 | Suzie's Food for Thought Cafe | 17 | Wasabi |
| 7 | Suzie's Blair School of Music | 18 | Zeppos Dining |
| 8 | Suzie's MRB III | 19 | Rothschild Dining Center |
| 9 | Suzie's Featheringill | 20 | Café Carmichael |
| 10 | Holy Smokes Kosher Food Truck | 21 | West End Bagel Co Production |
| 11 | E. Bronson Ingram Dining Center | | |

### Coverage survey (2026-10-04)

Each unit was selected in a fresh session. For each one, the menus were counted and the first menu's items were fetched.

- **No unit has sub-locations.** `childUnitsPanel` was empty for all 21, so the unit → menu → items flow covers everything.
- **14 units have menus**, 178 in total, covering Sunday Oct 4 through Saturday Oct 10 (5–7 days ahead depending on the unit):

  | Unit | Menus | First menu (items) |
  |---|---|---|
  | Rand (1) | 12 | Breakfast (68) |
  | Commons (2) | 28 | Breakfast (30) |
  | Kissam (3) | 17 | Breakfast (19) |
  | Pub at Overcup Oak (4) | 6 | Daily Offerings (31) |
  | EBI (11) | 12 | Breakfast (24) |
  | Commons Munchie (12) | 7 | Daily Offerings (22) |
  | Kissam Munchie (13) | 7 | Daily Offerings (21) |
  | Highland Munchie (14) | 7 | Daily Offerings (21) |
  | Highland Hotline (15) | 17 | Breakfast (20) |
  | Local Java (16) | 6 | Daily Offerings (32) |
  | Zeppos (18) | 9 | Lunch (49) |
  | Rothschild (19) | 12 | Brunch (36) |
  | Café Carmichael (20) | 17 | Brunch (17) |
  | West End Bagel Co (21) | 5 | Daily Offerings (25) |

- **7 units show "There are no menus available for this location."** These are VandyBlenz (5), the four Suzie's (6–9), Holy Smokes (10) and Wasabi (17). It's unknown whether this is permanent or just this week.

## Ways to get the data

1. **Call the endpoints from a script (recommended).** Use Python with `requests.Session()` and BeautifulSoup. Run it once a day on a schedule and write the results to JSON or SQLite. It's fast and needs no browser.
2. **Use only the menu pages for names, and add nutrients separately.** Ingredient search only needs dish names (see below), which come from step 2 alone. Nutrient amounts need one request per dish either way, through the labels (3) or through `SelectItem` plus the grid (3b). There's a `Menu/DeSelectAllItems` endpoint but no "select all".
3. **Headless browser (Playwright).** It's slower and breaks more easily. Only use it if the site adds bot protection.
4. **Ask Campus Dining or CBORD for a data feed or export.** This is the cleanest option, given the robots.txt point below.
5. **Reuse existing work.** CBORD NetNutrition runs at many universities, so search GitHub for "netnutrition scraper".

## Constraints and open questions

- **The site disallows all crawling.** `https://netnutrition.cbord.com/robots.txt` says `User-agent: *` / `Disallow: /`. Contact Campus Dining before deploying. In any case, scrape once a day at about 1 request per second, with a user agent that identifies the project and gives contact info.
- **"Ingredient" search means matching dish names**, e.g. "chicken" finds "Jamaican Jerk Chicken". It doesn't use the label's ingredient list. Things to watch for:
  - Match whole words, or "rice" will match "Licorice".
  - Handle plurals ("mushroom" vs "Mushrooms").
  - Names often don't say what's in them: "Beef Taco Meat" also contains eggplant and cauliflower, and "Pad Thai" won't match "chicken". Including the station name in the searchable text helps a little.
- **Only a few micronutrients are available:** Vit A, C and D, calcium, iron and potassium. For nutrients that aren't there, like zinc, the plan is a separate static page listing foods for each nutrient (e.g. zinc: beef, mushrooms). Users then search those foods by name. That page doesn't need any scraping.
- **Request volume:**
  - **Names only:** about 14 unit requests plus about 25–30 menus a day, or about 200 requests to pull the whole posted week at once.
  - **With nutrient amounts:** add one request per dish, which is several hundred to a few thousand a day. Dishes repeat often, so cache by item name and serving size. Whether `detailOid` stays stable across days hasn't been checked.
- **"Open right now":** the home page's Open/Closed badges and the hours popup exist, but where the hours data comes from hasn't been investigated.
