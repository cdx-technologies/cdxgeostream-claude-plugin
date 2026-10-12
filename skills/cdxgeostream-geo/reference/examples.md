# Worked examples

## 1. Enrich a column of ZIP codes
User pastes a column of ZIPs and wants city/state + population.
→ One call: `zip_lookup` with `zips` = the whole column. Map `results` back to the rows; for any
`{"status": "not_found"}` entry, leave that row blank and note it.

## 2. Find prospects near a store, then size the market
"Which ZIPs are within 25 miles of 80401, and what's the population of each?"
→ `radius` with `center="80401"`, `radius=25` → take the returned ZIP list →
`demographics` with `zips` = that list (one call). Sum the populations.

## 3. Geocode addresses for mapping
"Get lat/lng for these 40 addresses."
→ One `geocode` call with all 40 in `addresses`. (Costs 4 credits each.) Rows where
`status != "matched"` couldn't be geocoded — report them.

## 4. Optimize a delivery run
"Best order to visit these 8 stops starting from the depot?"
→ `optimize_route` with `stops` = [depot, ...the 7 others], `round_trip=true` for a closed loop.
Use `method="straight"` (1 credit per stop) unless the user needs true road distances, then
`method="aco"` (3 per stop — the stops are ordered by straight-line distance and the real road legs measured).
`method="here"` (4) orders the same legs on real road travel. If you get a "too large" error,
the stop cap was exceeded.

## 5. Clean a mailing list
"Validate these mailing addresses and give me ZIP+4."
→ `verify_address` with `addresses` = `[{address, city, state, zip}, ...]` (the whole list goes in one
call). Use the standardized fields + ZIP+4 from each result; flag any not deliverable.

## 6. Mixed bulk job in one round-trip
Need several different ops over a large dataset?
→ `batch` with `operations` = e.g.
`[{"op":"zip","args":["80401"]}, {"op":"distance","args":["10001","94105"]}, {"op":"geocode","args":["90210"]}]`.
Each entry in `results` is `{"result": ...}` or `{"error": ..., "status": ...}`, in order.

## 7. Check budget before a big job
Before geocoding 5,000 addresses, call `account_balance` and read `credits_available`. 5,000 addresses
× 4 credits = 20,000 credits — if `credits_available` is lower, tell the user to top up first.
