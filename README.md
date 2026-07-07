# recross

Real-time failure/revert tracker comparing [Across Protocol](https://across.to) and [Relay](https://relay.link).

A single-file, dependency-free HTML app (`index.html`) that shows the cross-chain
transfer **failure rate** for each protocol (plus underlying counts and a
per-chain breakdown), with a 1h/6h/24h lookback toggle and 2-minute auto-refresh.

## Methodology

Both cards show a rate — failed transfers over transfers whose outcome is
known — never a bare count, so the two protocols are comparable regardless of
volume. Transfers still in flight are excluded from the denominator, and
anything that can't be verified is reported separately instead of being
silently counted (or dropped).

### Relay

- **Failures** — requests the `api.relay.link/requests/v2` API marks
  `status=failure` **or** `status=refund` (a refund is only issued when the
  transfer could not complete), paginated over the full window via
  `continuation`. If the page cap is hit the count is shown as `N+`.
- **Rate** — an unfiltered query over the same window gives the denominator:
  completed = `success + failure + refund`. In-flight requests
  (`waiting`/`pending`/`submitted`) are excluded. If the window has more
  requests than the sample cap, the rate is computed over the most recent
  sample and labeled as such.

### Across

- **Deposits** — `FundsDeposited` events scanned from every EVM SpokePool via
  public RPCs. SpokePool addresses are resolved live from the HubPool registry
  (`crossChainContracts`) on Ethereum — nothing hardcoded to go stale. Both
  event generations (`V3FundsDeposited` and the post-migration bytes32
  `FundsDeposited`) are recognized. The scan window is located by block
  timestamp search, not a fixed blocks-per-hour estimate, and deposits are
  filtered by their `quoteTimestamp`.
- **Outcomes** — deposits are matched against `FilledRelay` / slow-fill events
  on their destination chain. A deposit only counts once its **fill deadline**
  has passed ("decided"); after the deadline a fill is impossible on-chain, so
  unmatched = failed (it will be expired/refunded). Apparent failures are
  double-checked against the `deposit/status` API. Deposits whose deadline
  hasn't passed are "in flight"; deposits that can't be verified (e.g. non-EVM
  destination and status API unreachable) are excluded from the rate and
  surfaced as "unverifiable".
- **Coverage caveats are visible** — chains whose RPC is unreachable or whose
  scan window was truncated are listed on the card instead of silently
  reporting zero.

## Running

No build step — serve the repo statically and open `index.html`:

```sh
python3 -m http.server
```

Works as-is on any static host (GitHub Pages, Vercel, etc.).
