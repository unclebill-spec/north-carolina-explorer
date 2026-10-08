# North Carolina Explorer — agent handoff

Static Leaflet map for a nurse household (Bill and Heather, RNs) plus the Anna engineer-family profile, the same shared app as the other 10 Explorers
(`app.js`, `areas.js`, `perm.js`, `profiles.js`, `extras.js`, `style.css`, `anna.js`, `anna_build.py` are byte-identical copies of `/workspace/kentucky/explorer/*`;
run `scripts/sync_shared.sh` right before every build/publish and log any shared-file edit in KY `explorer/AGENTS.md`). `sw.js` differs only in its cache prefix (`ncx-`).
NC settings live in `explorer/build.py` `STATE`.

Live: https://unclebill-spec.github.io/north-carolina-explorer/ · repo unclebill-spec/north-carolina-explorer (created by publish.sh: `gh repo create --public` + Pages API) ·
progress log `/workspace/north-carolina/PROGRESS.md`, status `/workspace/north-carolina/STATUS.md` (newest first, ET).
Copied from the Utah code base Oct 5 2026. Scripts pick the state from the folder (`scripts/common.py` ST NC, fips 37, 100 counties).

## Caps (same as Kentucky / Tennessee)
5+ acres 2bd/2ba $300k–$500k; 1+ acre 3bd/2ba ≤ $425k; near-hospital 1,600+ sqft 3bd/2ba ≤ $325k (townhomes/condos OK, good condition, ≤ 10 min free-flow to a hospital with a 10+ bed ER).
Big land: 50+ ac under $250k. Cave/falls on the property up to $500k. Anna: 2+ bd / 2+ ba up to $550k.

## Zillow is blocked (HTTP 403 to the box, Oct 5 2026) -> Redfin
Never work around it. Homes come from Redfin's public map-search data:
- `scripts/rfsearch.py` -> `data/rfsearch/<County>.json` (every active listing per county; region_type=5, NC county ids 2007–2106); Wake + Mecklenburg hit the 5,000 cap so
  `scripts/rfzip.py` splits them by ZIP -> `data/rfsearch_zip/` (10 ZIPs answered HTTP 202 empty = bot check; not retried around).
- `scripts/listings_build_rf.py` -> `listings.json` + `listing-photos/` (OSRM drives cache `data/rfsearch_drives_cache.json`, listing-page cache `data/rfsearch_detail_cache.json`;
  CAP 160/160/120, PER_COUNTY 6; near-hospital condition screen uses the full Redfin description).
- `scripts/redfin_extras.py` (run after listings) -> `bigland.json`, `cavefalls.json` (cave/falls keyword prefilter on the search remarks, then full description).
- bargains.py / top_lists.py comps: Redfin active listings (data/rfsearch*).

## Data pipeline (from /workspace/north-carolina; pandas scripts use /workspace/kentucky/.venv/bin/python, the rest /usr/bin/python3)
1. `scripts/hospitals_research.py` -> `data/hospitals.json` (CMS + NC trauma designations); `scripts/layers.py` (venv) -> block CSVs, schools, SEDA, RN wages
   (Durham-Chapel Hill RN wage suppressed by BLS -> Chatham/Durham/Orange/Person use the NC statewide median, noted in `data/statewide/raw/oews_area_of.json`).
2. `scripts/appeal_build.py`; `scripts/er_beds.py` (103 acute, 100 qualify; estimate rule).
3. Climate: `climate/raw` (837 NOAA stations extracted from the KY national tarballs), `climate/parse.py`, `climate/build_clim.py` (BOX NC). Activities: `data/osm/wd_act.py` + `wp_cat_act.py`.
4. Compare areas: `compare/areas_build.py` (venv; Charlotte, Raleigh, Greensboro, Durham, Winston-Salem; land medians from Redfin, not Zillow).
5. Homes: see Redfin above.
6. Perm RN jobs: `scripts/perm_jobs.py` -> `data/perm_jobs.json` (classification from the shared `/workspace/kentucky/scripts/perm_jobs_common.py`: cardiac cath lab hidden,
   cath recovery / step-down / holding kept). Sources: Atrium (Advocate Workday, NC facet), Novant / WakeMed / FirstHealth (Jibe), Duke (SuccessFactors), ECU (Phenom),
   Cone + Cape Fear Valley (Workday), CaroMont (HealthcareSource), Lifepoint ORC; UNC Health = Cloudflare -> `data/unc_websearch.json`; HCA Mission LAST: careers site 403 ->
   `data/hca_websearch.json` (web search).
7. Travel RN jobs: `/workspace/tj_nc` (Vivian + Advantis NC pages) then `scripts/travel_jobs.py` (NC ALIAS) -> `data/travel_jobs.json`. Needs explorer/data/data.js.
   NC's `scripts/travel_filters.py` keeps cath recovery / step-down / holding (CATH_KEEP) and drops cath lab.
8. Airports `data/airports/airports.py`; attractions `data/attractions/wd2.py` + `make_attractions.py`; thumbnails `explorer/fetch_thumbs.py` (after a build).
   `data/forsale/*` (Crexi) were NOT run for NC — they still point at /workspace/utah; don't run them without fixing OUT/PH first.
9. Border: `/workspace/border/scripts/static.py NC`, `make_border.py NC` -> copy `/workspace/border/out/NC.json` to `explorer/border.json` (TN is a covered map; GA/SC/VA static).
10. Ski + peaks: `/workspace/mtn/scripts/make_state.py NC` (NC stations only for NC).
11. Anna: `/workspace/anna/scripts/eng_compile_nc.py`, `spouse_compile_nc.py`, `anna_homes_nc.py` (Redfin), `make_state_json_nc.py` -> `/workspace/north-carolina/anna.json`.
12. Publish: `sh scripts/sync_shared.sh && cd publish && PATH=/usr/bin:$PATH ./publish.sh -m "msg"` (flock /tmp/ncx_publish.lock, secscan dist + history, push, waits for Pages).
13. Tests: `perf/smoke.py BASE TAG`, `perf/test_homes.py`, `perf/test_perm.py`.

## Data checks added Oct 5 2026 (keep them)
- `scripts/listings_build_rf.py` norm(): a Redfin lot of >2,000 ac at <$200/ac is lot square feet typed into the acres field -> read as square feet.
- `scripts/redfin_extras.py`: big land dedupes (town, price, acres); cave/falls drops a listing when every quote puts the falls off the property (nearby, community, park, "N remote waterfalls"...).
- Anna tests: `/workspace/anna/test_anna.py <NC URL> 36.10,-80.24` (Winston-Salem zoom with engineering pins).

## Known gaps (Oct 5 2026)
- Zillow answers 403 to this computer (not bypassed) -> all homes, big land, cave/falls, bargain comps and Anna homes are Redfin. Ten Wake / Mecklenburg ZIP searches came back empty (Redfin bot check).
- Cave/falls: keyword match on Redfin remarks; 79 waterfalls, 0 caves (the only cave mention was off-property).
- Big land: 11 lots, all raw land (none with a home under $250k at 50+ ac).
- Durham RN wage suppressed in BLS -> statewide median used. UNC Health (Cloudflare) and HCA Mission (403) perm jobs are web-search samples (17 + 5).
- Anna: 22 of 24 engineering salaries are BLS OEWS estimates (NC has no pay-transparency law); bonus amounts estimated. Not searched: Spirit AeroSystems Kinston, Honda Aircraft, HAECO, Siemens Energy.
- Peaks missing: Chimney Rock, Graybeard, Hawksbill, Shortoff (no article); Mount Craig, Pisgah, Elk Knob (no coords). Carowinds geocodes to SC.
- Crexi businesses / buildings not built for NC. Some travel facilities unmatched (Reputable Healthcare, HealthTrust Hickory, Kindred, Signature); most Vivian posts don't name the facility.

## Target stores layer (Oct 7, 2026 ~5:15 PM ET, Target worker) — shared app.js + new shared target_build.py + build.py hook
- Bill: a Target stores layer on all 11 maps. Every Target in the state + stores within ~15 mi outside the state line (border rule: `bst`/`bco`/`bmi`, no `county` field, so never in county stats).
- **Data:** `/workspace/target/` (log: PROGRESS.md). Source = Target's own store directory (`target.com/store-locator/store-directory/<state>`, the official per-state list and count) + each store's page (`target.com/sl/<slug>/<id>`: JSON-LD geo, address, regular hours, phone, services). OpenStreetMap (Overpass, brand:wikidata=Q1046951) is only used to find neighbor-state stores near the line and as a count cross-check. Scripts: `scripts/fetch_dir.py` → `fetch_sl.py <STs>` → `parse_sl.py` → `make_target.py <STs>` (→ `out/<ST>.json`, copy to `<state>/explorer/target.json`) → build → `scripts/drives.py <statedir>` (OSRM free-flow minutes from each listing to its fastest of the 3 nearest stores, written into target.json `drv`, cache `cache/osrm.json`) → build again.
- **Build:** `explorer/target_build.py` (shared, md5-identical everywhere; in each sync_shared.sh list). build.py line right after `border_build.add(data, HERE)`: `import target_build; target_build.add(data)` (re-apply: `/workspace/target/patch/hook_build.py <statedir>`). Writes `K.targets` (whole rows in core.js, ~250 B each) + `K.meta.target`, and `nearby.tg` on every property (OSRM minutes when `drv` matches the listing id + spot, else straight-line × 1.3 "approx"). No target.json → no-op.
- **App (block "Target stores" before `function kyxMtn`; re-apply `/workspace/target/patch/patch_app.py app.js`, idempotent, marker `function kyxTarget(`):** icon `G.tgt` (small red bullseye), layer key `tgt` (on by default, icons from `FULL.tgt` = 9, red faint dots below, grouped like the other kinds), right-stack button "Target" (also in the landscape wheel; shows in Bill's P1–P4 and Anna mode — profiles don't filter it), Layers row, Map key row, search ("Target …"), card `R.target` (name, address, phone, store number, regular hours, services, target.com link; border stores get the 🧭 state tag), share `#target=<store number>` (index.html#…; no share page), Back stack like every card, property cards' Nearby row "🎯 Target · ~N min drive". `BST_NAME` gained the western states (border tags now say "South Dakota" instead of "SD").
- **Test:** `/usr/bin/python3 /workspace/target/test_target.py BASE TAG [anna]` (412×915 touch + 1280×720: button, solo on/off, zoom tiers + groups, pin tap, share links in-state + border, Nearby link + ‹ Back, Bill profile, Anna mode when the map has it, wheel over the column, 3 whole pills, no console errors). Screenshots `/workspace/target/shots/`.
- **Refresh:** Target opens/closes stores rarely; rerun the scripts above (fetch_sl.py only downloads pages not in cache/sl/; delete a page to refetch it).

## Active filter on top: kyxFoc (Oct 7, 2026 ~9 PM ET) — shared app.js/style.css
- The active right-side button / Anna button / open Top 10 list draws its pins 1.4x larger (groups 1.15x) and above everything; other pins (trauma, airports, cities...) go 0.72x and underneath while it is on. No filter = unchanged. Details: /workspace/kentucky/explorer/AGENTS.md "Active filter on top"; test /workspace/filterfocus/smoke.py BASE TAG.
- Oct 8, 2026: "Hospitals" right-side button (all hospitals incl. border, existing icons; trauma centers topmost; hidden in Anna). Details: KY explorer/AGENTS.md "Hospitals button: kyxHosp".
