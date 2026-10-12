# CDXGeoStream for Claude

Live geographic data for the United States and Canada, in Claude. Ask in plain language and Claude
calls the CDXGeoStream tools to:

- look up ZIP and postal codes (city, county, state or province, coordinates, population);
- measure straight-line distance, list the ZIPs within a radius, or find the nearest ZIPs;
- find ZIPs by city, county or state;
- pull U.S. Census demographics for a ZIP;
- geocode addresses, reverse-geocode coordinates, and look up U.S. Census tracts;
- get driving distance and time, and order a list of stops into an efficient route;
- validate and standardize U.S. mailing addresses to USPS form (ZIP+4, deliverability).

## What's in this plugin

- **The CDXGeoStream connector** (`.mcp.json`): one remote MCP server,
  `https://mcp.cdxgeostream.com/mcp`. Its 16 tools are all read-only.
- **The `cdxgeostream-geo` skill** (`skills/cdxgeostream-geo/`): instructions that help Claude pick
  the right tool, send whole lists in one call, keep costs down, and recover from errors.

The plugin runs no local code and installs no packages. The only service it contacts is
`mcp.cdxgeostream.com`: Claude connects there to sign you in and list the tools, and each tool call
sends the inputs you asked about — ZIP or postal codes, addresses, or coordinates — and gets the
result back. The sign-in page itself is served from `mcp.cdxgeostream.com` and shows our logo from
`app.cdxgeostream.com`.

## Before you start

You need a CDXGeoStream account and API key. Sign up at [cdxgeostream.com](https://cdxgeostream.com)
— the Free plan covers ZIP lookups, distances, demographics, census tracts by ZIP and address
verification; Pro, Enterprise or purchased credits add geocoding, driving routes and road route
ordering. Every tool call uses credits from your account; the `pricing` and `account_balance`
tools are free.

## Signing in

The first time Claude uses the connector, it opens the CDXGeoStream sign-in page. Paste your
`cdx_live_` API key there to authorize Claude. Claude receives a token tied to that key — never the key
itself — and revoking the key in your CDXGeoStream account disconnects it immediately.

## Privacy and support

- Privacy policy: <https://app.cdxgeostream.com/privacy/>
- Setup guide: <https://cdxgeostream.com/mcp>
- Support: <https://cdxgeostream.com/support>
