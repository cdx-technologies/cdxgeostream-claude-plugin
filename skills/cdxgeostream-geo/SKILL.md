---
name: cdxgeostream-geo
description: >-
  ZIP/postal lookup, distance & radius, demographics, geocoding, reverse geocoding, census tracts,
  driving routes, route optimization, and USPS address verification for the US and Canada — via the
  CDXGeoStream geo MCP tools. Use when the user asks to look up a ZIP or postal code, compute the
  distance between locations, find ZIPs near a point or within a city/county, geocode addresses to
  latitude/longitude, reverse-geocode coordinates, get census tract codes or demographics (population,
  income, age), plan or optimize a multi-stop driving route, or validate/standardize mailing addresses.
---

# CDXGeoStream geo tools

These tools wrap the CDXGeoStream geo API (US + Canada). They are provided by the **`cdxgeostream-geo`
MCP connector** — this Skill only tells you *how and when* to use them. Always call the tools; never
try to reach the API directly over the network (it won't work on every surface, and the connector
already carries the API key and billing).

## Choosing a tool

- **ZIP / postal code → place** (city, state/province, county, lat/lng, population): `zip_lookup`
- **Distance between two places** (by ZIP/postal code): `distance` (great-circle / straight-line)
- **Driving distance & time**: `route` (real road routing)
- **ZIPs near a point**: `radius` (all within X miles) or `closest` (the nearest N)
- **Which of these ZIPs is nearest to X**: `closest_in_list`
- **ZIPs in a city/county/state**: `find`
- **Address → coordinates**: `geocode`; **coordinates → place**: `reverse_geocode`
- **Census tract (FIPS/GEOID)**: `census_tract`
- **Population / income / age / housing**: `demographics` (by ZIP); `area_demographics` for a whole
  city, county or state
- **Best order to visit several stops**: `optimize_route`
- **Is this a real, deliverable mailing address (ZIP+4)**: `verify_address`
- **Many of the above at once over a big list**: `batch`
- **What does an operation cost / what plan am I on**: `pricing`, `account_balance` (neither costs credits)

See `reference/functions.md` for the full parameter list of every tool.

## Inputs

- ZIP/postal codes are strings (`"10001"`, Canadian `"M5H 2N2"`). Distances take `unit` =
  `miles` | `km` | `nautical` (default miles).
- `geocode` takes street addresses or ZIPs; `reverse_geocode` and `closest` take `[lat, lng]`.
- `route`/`optimize_route` accept addresses, ZIPs, or `"lat,lng"` strings.
- **Say when a result is approximate.** `distance` and `optimize_route` with `method: straight`
  place a street address at its ZIP or postal code's center. `distance` lists those inputs in `approx`;
  `optimize_route` marks each such stop `approx: true`, and stops sharing a code come back 0 mi apart.
  Tell the user those figures are approximate, and why.
- **A ZIP's census tract is the tract at the ZIP's center.** A ZIP covers several tracts, so
  `census_tract` on a ZIP returns `basis: "zip_center"` and `approx: true`. Say so; when the exact tract
  matters (eligibility, a client deliverable), ask for the street address or `"lat,lng"`.
- **Straight-line routes have no travel time.** `optimize_route` with `method: straight` returns
  `travel: "straight_line"` and null `duration_*` fields. Quote its miles as straight-line, never as
  driving; use `aco` or `here` for drive times.
- **Say whether a suite was checked.** Each `verify_address` result carries `secondary_status`. Tell the
  user when it is `unconfirmed` (the unit given is not a known unit there) or `missing` (the building
  needs one and none was given). `not_checked` means the unit could not be checked — mention it only
  when the user gave a unit, and say that unit is unverified.
- **Batch, don't loop.** Every list-accepting tool takes many inputs in one call. When the user gives
  you a column or list of values, pass them in as few calls as the plan allows (or use `batch`) — it's
  far fewer round-trips. One call takes up to the plan's per-call limit (`pricing` →
  `caps.<plan>.call_max_items`; 500 on Free and 1,000 on Pro as shipped): split a longer list into
  calls of that size. For most tools the credit cost is the same as separate calls; **`distance` is the
  exception** — send every pair to the `distance` tool in one call (1 credit per 25 pairs), never
  through `batch`, where each pair is its own 1-credit op.

## Credits & cost awareness

Operations are credit-metered against the account's plan. Each successful result includes a
`credits_remaining` field (the account's live balance) and `credits_charged` (this call alone). On an
unlimited plan `credits_remaining` is `null` and `credits_charged` is left out.

**What each plan allows** is stated in each tool's description. On the free plan, `geocode` takes ZIP and
postal codes, and `census_tract` takes those plus `"lat,lng"` points; anything else sent to them (a
street address, or coordinates sent to `geocode`), and all of `reverse_geocode`, `route` and the road
`optimize_route` methods, need a paid plan or purchased credits.

- `zip_lookup`/`demographics` cost 1 credit **per item**; `distance` costs 1 credit **per 25
  pairs** in a call (minimum 1, at most 10,000 pairs), so send pairs together; geocoding, reverse geocoding,
  cost 4 (each is a live lookup per item); census tracts and address verification cost 2;
  routing costs 3 per pair **plus the
  `geocode` rate (4) for each endpoint given as a street address** rather than a ZIP, postal code or
  `lat,lng` — so an address-to-address route is 3 + 4 + 4 = 11, not 3.
- `optimize_route` costs `stops × method weight`: `straight` ×1, `aco` ×3, `here` ×4. 25 stops =
  25 / 75 / 100 credits. Both road methods return the same real road legs; they differ only in how
  the stops are ORDERED — `aco` orders them by straight-line distance (fewer lookups, so cheaper), `here` orders
  them on real road travel. **Prefer `aco` for road routing**; reach for `here` only if the caller
  specifically wants the road-optimized order. Stop caps are 100 for both.
- `radius`/`closest`/`find` cost 1 credit **per 25 rows returned**, rounded up, minimum 1 — 3 ZIPs
  costs 1 credit, 500 costs 20, 5,000 costs 200. Pass `limit` to cap what you get back and therefore
  what you pay. `closest_in_list` costs 1 credit **per 25 CANDIDATES you send**, rounded up, minimum 1
  — its output is fixed at 3 rows, but you are buying the search, and the search is as big as the list
  (100 candidates = 4 credits, 1,000 = 40). `radius` also
  has a per-plan reach cap in miles and `413`s above it. See `reference/pricing.md`.
- Before a **large** geocode / route / optimize job, you may call `account_balance` to confirm there
  are enough credits.
- If a tool fails with an **out-of-credits** error, tell the user they're out of credits and need to
  top up or upgrade their plan — **do not retry**, it will keep failing. "Not enough credits for this
  request" is different: some credits remain, so a smaller request can still run.

## Error recovery

See `reference/errors.md`. In short:

- **Too large / cap exceeded** (e.g. too many optimize stops or batch ops): split the input into
  smaller chunks and call again.
- **Radius over the plan's reach cap** ("The Free plan's radius limit is 250 miles. Use a smaller
  radius, …"): splitting doesn't help — re-run with a radius within the limit, or tell the user to
  upgrade.
- **Out of credits**: stop and tell the user to top up; do not retry. **Not enough credits for this
  request**: offer a smaller request, or a top-up.
- **Invalid input** (names a field): fix that field and call again.
- **Temporary problem**: retry once after about a minute; check `account_balance` before re-running a
  large job, because a failed call can still have been charged.
- **Rate limited**: wait ~1 second and retry.
- **Per-item not found**: list/batch tools return a `{"status": "not_found"}` entry for the bad
  item and still succeed for the rest — report which inputs weren't found and continue.
- **Not in the plan**: the plan doesn't include that feature; a purchased top-up or an upgrade
  unlocks it. Tell the user. `geocode` and `census_tract` refuse only the inputs that need it and run
  the rest; `reverse_geocode` and `route` return every row `upgrade_required`; road `optimize_route`
  refuses the call. The message names the feature and why.
- **Batch arg errors** ("demographic needs [zip, field]; got 1 argument"): add the missing args — the
  `batch` description lists every op's positional args.
