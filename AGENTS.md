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
