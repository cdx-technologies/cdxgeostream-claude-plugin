# Error handling & recovery

The geo tools surface failures as MCP tool errors with an actionable message. How to react:

| Situation | What the message says | What to do |
|-----------|----------------------|------------|
| **Out of credits** | "Out of credits — the account's monthly grant and prepaid top-up balance are exhausted …" | Stop. Tell the user they're out of credits and need to top up / upgrade at cdxgeostream.com/account. **Do not retry** — it will keep failing. |
| **Not enough credits for this request** | "Not enough credits for this request — it needs N and the account has M …" | The account has some credits, just not N. Tell the user; offer a smaller request that fits in M (fewer items, a lower `limit`), or topping up. |
| **Request too large** | "Request too large: this method is limited to N stops / batch exceeds max N operations" | Split the input into chunks ≤ the stated limit and call again per chunk. |
| **Radius over the plan's reach cap** | "Request too large: radius exceeds the N-mile per-call limit for your plan" | Splitting does **not** help — the cap is on the radius itself, so ignore the generic "split the input" tail on that message. Re-run with a radius ≤ N (the cap is measured in miles whichever unit you sent), or tell the user to upgrade the plan. Inside `batch` only that one item fails. |
| **Not in the plan** | "This feature isn't included in the account's plan (upgrade required for …)" | The current plan doesn't include that feature (e.g. geocoding a street address, road route optimization). A purchased credit top-up unlocks it, as does upgrading. Tell the user; offer the straight-line `optimize_route` method where relevant. |
| **Rate limited** | "Rate limited … Wait ~1 second and retry." | Brief pause, then retry the same call. It can fire even at a modest request rate: a completed request also debits the geocoding and routing lookups it made, and that charge leaves a short deficit the next request waits out. So this does **not** mean you're sending too many requests — don't split the work or shrink the call. |
| **Invalid input** | "Invalid input: pairs.0.to: Field required. Correct it and call again." | The named field is missing or malformed. Fix that field and call again — retrying unchanged will fail the same way. |
| **Temporary problem** | "CDXGeoStream couldn't complete this request because of a temporary problem on our side …" | Retry once after about a minute. A failed call can still have been charged, so before re-running a large metered job, check `account_balance`. If it fails again, tell the user and point them to support. |
| **Authentication failed** | "Authentication failed — the API key is missing/invalid/revoked …" | The connector's API key is wrong or the account is suspended. Tell the user to check the connector configuration; don't retry blindly. |

## Per-item not found (not an error)

List and `batch` tools do **not** fail the whole call when one input can't be resolved. Instead the
result array contains a per-item marker:

- `zip_lookup` / lookups: `{"zip": "00000", "status": "not_found"}`
- `distance`: `null` for an unmatched pair
- `batch`: `{"error": "...", "status": "not_found"}` for that op, with the others succeeding

Report which inputs weren't found and continue using the results that did resolve.
