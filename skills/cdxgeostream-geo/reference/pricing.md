# Credits & pricing

Authoritative, live rates are returned by the **`pricing`** tool (it reflects any portal overrides).
The table below is the code default as a quick reference — when in doubt, call `pricing`.

## Credits per operation

Two different billing shapes — read the middle column, it matters:

| Operation | Billed | Credits |
|-----------|--------|---------|
| `zip_lookup` | per item | 1 per item |
| `distance` | per PAIR sent, per call | ceil(pairs ÷ 25), min 1; at most 10,000 pairs per call |
| `demographics` (and citydata/countydata/statedata via `batch`) | per item | 1 per item |
| `geocode`, `reverse_geocode` | per item | 4 per item |
| `census_tract`, `verify_address` | per item | 2 per item |
| `route` | per pair | 3 per pair (+ the `geocode` rate for each endpoint given as a street address rather than a ZIP / postal code / `lat,lng`) |
| `optimize_route` | per stop | `straight` stops×1, `aco` stops×3, `here` stops×4 |
| **`radius`, `closest`, `find`** | **per ROW returned** | **ceil(rows ÷ 25), min 1** |
| `closest_in_list` | per CANDIDATE sent | ceil(candidates ÷ 25), min 1 |
| `pricing`, `account_balance` | — | 0 (free) |

`pricing` also lists `route_grid`: the REST endpoint `POST /v1/route-grid` (not an MCP tool), a
driving distance grid billed by size — 100 credits for each entry in the longer list once both lists
have 5 or more, so 500 × 400 is 50,000 credits. For grids through these tools,
use `route` pairs.

**Per-item ops:** a "unit" is one item in the list (one ZIP, one address, one route pair, one stop).
Passing a list of N items costs N × the per-item rate — the same as N separate calls, in one round-trip.

**`distance` is not per-item.** One call is billed on the pairs it carries, 1 credit per 25, so 500
pairs in one `distance` call cost 20 — while 500 `distance` ops in `batch` cost 500, because each op
is its own one-pair call. Send pairs to `distance`, not `batch`.

**`optimize_route` methods.** All three are billed `stops × weight`: `straight` ×1 (local, no road
data), `aco` ×3 (the stops are ordered by straight-line distance, then the road legs are measured), `here` ×4 (the
stops are ordered on real road travel, one extra lookup on top of the same legs). At 25 stops that is
25 / 75 / 100 credits.

The two road methods return the SAME real road distances and times — they differ only in how the
stops are ordered. `aco` makes strictly fewer lookups, which is why it costs less. Prefer `aco` for
road routing unless the caller specifically wants the road-optimized order.

Stop caps are **100** for both road methods, and both `413` above that. Read `pricing` for live
figures rather than hardcoding these — an operator can retune the weights and the caps.

**Per-row ops** (`radius`, `closest`, `find`) cost **1 credit per 25 rows returned**, rounded up, with
a 1-credit minimum. A sweep returning 3 ZIPs costs 1 credit; 500 ZIPs costs 20; 5,000 costs 200. What
you are charged for is what comes back — the entered radius / `n` / `limit` only matters through the
row count it produces. That is still a 25× discount on looking the same ZIPs up individually
(`zip_lookup` is 1 credit each), which is the point of the list form.

To control cost, pass `limit`: it means "up to N rows", and you are billed for the rows you actually
receive. The `radius` and `find` tools, called with no `limit`, return a short default list
(`caps.<plan>.mcp_default_rows`, 100 unless changed) rather than the whole allowance. A radius or find
op inside `batch` has no such default: with no limit it returns, and bills, the whole allowance. The plan's row allowance (`caps.list_max_rows`: free 500, pro 5,000, enterprise 25,000)
CLAMPS the result and reports `limit` + `truncated` rather than erroring. If your balance can't cover
the full allowance you get fewer rows, not an error — again reported via `limit` + `truncated`.

`closest_in_list` is priced on the CANDIDATE LIST you send, not the rows it returns: ceil(candidates ÷
25), minimum 1. Its output is fixed at 3 rows, but the output is not the deliverable — what you buy is
the search, and the search is exactly as big as the list. 100 candidates costs 4, 1,000 costs 40.
Pricing it per call made it a 1,000× arbitrage on the same question asked with `distance`.

`radius` additionally has a **reach cap** (`caps.radius_max_miles`: free 250, pro 1,000, enterprise
3,000). Unlike the row allowance this one REJECTS with `413` — quietly answering a 3,000-mile question
with a 250-mile result would be a wrong answer. The cap is in miles whatever unit you send. Inside a
`batch`, an over-cap `radius` fails only that item.

Call `pricing` for the live divisor rather than hardcoding 25: `rows_per_credit` carries it, and
`row_priced_ops` names the ops it applies to.

## Plans

Credits are pooled per **account** (all of an account's API keys draw from one shared balance).
An account has a monthly credit grant (by tier) plus an optional non-expiring prepaid top-up balance,
drawn after the monthly grant is spent. Default monthly grants:

| Tier | Monthly credits |
|------|-----------------|
| free | 1,000 |
| pro | 100,000 |
| enterprise | 600,000 |

When the monthly grant **and** the top-up balance are both exhausted, metered calls hard-block with an
out-of-credits error. Call `account_balance` to see `credits_available` (= monthly remaining + top-up).
Top up or change plans at https://cdxgeostream.com/account.
