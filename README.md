# recross

Real-time failure/revert tracker comparing [Across Protocol](https://across.to) and [Relay](https://relay.link).

A single-file, dependency-free HTML app (`index.html`) that shows total cross-chain
transfer failure counts for each protocol plus a per-chain breakdown, with a
1h/6h/24h lookback toggle and 90-second auto-refresh.

## Data sources

- **Relay** — the public `api.relay.link/requests/v2` REST API, filtered to
  `status=failure`.
- **Across** — `V3FundsDeposited` events scanned directly from SpokePool
  contracts via public RPCs, with past-deadline deposits verified against the
  `app.across.to/api/deposit/status` API (`expired` / `refunded` / `pending`
  count as failures).

## Running

No build step — serve the repo statically and open `index.html`:

```sh
python3 -m http.server
```

Works as-is on any static host (GitHub Pages, Vercel, etc.).
