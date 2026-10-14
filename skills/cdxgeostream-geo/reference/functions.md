# CDXGeoStream geo tools — full reference

All tools are read-only. Live-lookup tools (marked ⌁) query live address or road data. Every successful
result is a JSON object that also carries `credits_remaining` and `credits_charged` (`null` =
unlimited plan). List-accepting tools should be called with the whole list at once.

| Tool | Params | Returns | Cost |
|------|--------|---------|------|
| `zip_lookup` | `zips: string[]` | `{results: [{zip, city, state, county, lat, lng, population}]}` | 1 / zip |
| `distance` | `pairs: [from, to][]`, `unit?` (miles\|km\|nautical) | `{unit, results: number[]}` (null per unmatched pair) | ceil(pairs ÷ 25), min 1; ≤ 10,000 pairs |
| `radius` | `center: string`, `radius: number`, `unit?`, `limit?` | `{center, unit, count, limit, truncated, results: [{zip, distance}]}` | ceil(rows ÷ 25), min 1 |
| `closest` | `center?: string` or `lat?`/`lng?`, `n?` (=5), `unit?` | `{unit, count, limit, truncated, results: [{zip, distance}]}` | ceil(rows ÷ 25), min 1 |
| `closest_in_list` | `target: string`, `candidates: string[]`, `field?` | `{closest_zip, distance, ...}` | ceil(candidates ÷ 25), min 1 |
| `find` | `city?`, `county?`, `state?`, `limit?` | `{count, limit, truncated, results: [{zip, city, state}]}` | ceil(rows ÷ 25), min 1 |
| `demographics` | `zips: string[]` | `{results: [{...demographic fields}]}` | 1 / zip |
| `area_demographics` | `level` (city \| county \| state), `state`, `name?` (city or county), `fields: string[]` | `{level, name, state, results: [{field, result}]}` — counts summed, medians population-weighted | 1 / field |
| `geocode` ⌁ | `addresses: string[]`, `field?` | `{results: [{status, latitude, longitude, ...}]}` | 4 / address |
| `reverse_geocode` ⌁ | `points: [lat, lng][]`, `field?` | `{results: [{zip, city, state, ...}]}` | 4 / point |
| `census_tract` ⌁ | `locations: string[]`, `field?` (geoid) | `{results: [{geoid, basis, approx?, ...}]}` — `basis` is `address`, `point` or `zip_center` | 2 / location |
| `route` ⌁ | `pairs: {from, to}[]`, `calculation?` (distance_mi\|distance_km\|time), `travel?` (driving\|walking\|bicycling) | `{calculation, travel, results: [{distance_mi, duration_min, ...}]}` | 3 / pair + 4 per street-address endpoint |
| `optimize_route` ⌁ | `stops: string[]` (stops[0] = origin), `method?` (straight\|aco\|here), `round_trip?`, `fixed_start?`, `fixed_end?`, `optimize_by?` (distance\|time), `travel?`, `gpx?` | `{status, method, travel, order, stops: [{seq, input, lat, lng, approx}], legs, total: {distance_mi, duration_min, ...}, dropped}` — `straight` gives `travel: "straight_line"` and null `duration_*` | stops × weight |
| `verify_address` ⌁ | `addresses: {address, city, state, zip, secondary}[]` | `{results: [{...USPS-standardized, ZIP+4, match_status, match_code, secondary_status}]}` | 2 / address |
| `batch` ⌁ | `operations: {op, args}[]` | `{results: [{result}\|{error, status}]}` | summed per op |
| `pricing` | — | `{pricing, tiers, gating, caps}` | 0 |
| `account_balance` | — | `{tier, credits_used, credits_remaining, credit_balance, credits_available, ...}` | 0 |

## Notes

- **Input formats**: ZIP/postal codes are strings; Canadian postal codes look like `"M5H 2N2"`.
  `geocode` accepts street addresses or ZIPs. `route`/`optimize_route`/`census_tract` accept
  addresses, ZIPs, or `"lat,lng"` strings.
- **Row-priced ops**: `radius`, `closest`, and `find` are billed per row returned —
  `ceil(rows ÷ 25)`, minimum 1 credit — on the dedicated tools and inside `batch` alike. `limit` (and
  `n` on `closest`) is therefore the cost control: you pay for the rows you receive. Omit `limit` on
  the `radius`/`find` tools to get a short default list (`pricing` → `caps.<plan>.mcp_default_rows`,
  100 unless changed); pass a larger `limit` for more, up to the plan's row allowance
  (`caps.<plan>.list_max_rows`), which clamps rather than errors. **Inside `batch` there is no short
  default** — a radius or find op with no limit returns the whole allowance, so always give it one. The response reports the `limit` applied and whether it was
  `truncated` — when it is, say how many you are showing and offer to fetch more. If the balance can't cover
  the full allowance you get fewer rows, not an out-of-credits error. `closest_in_list` is measured on
  the CANDIDATES SENT instead — ceil(candidates ÷ 25), min 1 — so its cost IS knowable before the call,
  unlike the row-priced ops. Call `pricing` for the live `rows_per_credit` divisor and for
  `input_priced_ops` / `row_priced_ops`, which say which rule applies to which op.
- **`optimize_route` methods**: `straight` is included in every plan; `aco` and `here` need a paid
  plan or purchased credits (a free key without credits is refused them). `straight` orders in a
  straight line, `aco` and `here` both use real road distances and differ only in how the stops are
  ordered (`aco` = by straight-line distance, `here` = on real road travel, one extra lookup). The method sets the
  cost weight — `straight` 1, `aco` 3, `here` 4 credits per stop (`pricing` →
  `optimize_method_weight`), so the same 25 stops cost 25, 75, or 100. The number of stops is capped
  (see `pricing` → `caps`); on a "too large" error, split the stops.
- **`optimize_route` with `straight` and street addresses**: `straight` looks nothing up, so it
  places an address at the center of the ZIP or Canadian postal code it ends with. Such a stop comes
  back with `approx: true`, and stops sharing a code are 0 mi apart — tell the user the order is
  approximate. An address it cannot place that way — no state and ZIP at its end, or a ZIP we do not
  hold — is skipped and listed in `dropped`. For exact street points
  use `aco` or `here`.
- **`batch` ops**: `zip`, `distance`, `radius`, `closest`, `closest_in_list`, `find`, `demographic`,
  `citydata`, `countydata`, `statedata`, `geocode`, `revgeocode`, `tract`, `route`, `optimize`,
  `verify`. `args` is positional; the `batch` tool's description lists each op's args in order (e.g.
  `demographic [zip, field]`), and most ops sent too few say what they need (`verify`, `optimize` and
  `closest_in_list` accept a short list and answer it as best they can). The whole batch is checked
  against the credit balance up front; blocked ops (plan/feature) return per-item `upgrade_required`
  and are charged 0.
- **`census_tract` with a ZIP** answers with the one tract at the ZIP's center (`basis: "zip_center"`,
  `approx: true`). A ZIP covers several tracts, and an address in it may be in another; use the street
  address or `"lat,lng"` when the exact tract matters.
- **`verify_address`** verifies the whole list in a single call (efficient for mailing lists). Each
  result's `secondary_status` says whether a suite or unit was checked: `confirmed`, `unconfirmed` (not a
  known unit at that address), `missing` (the building needs one), `not_required`, or `not_checked`.
