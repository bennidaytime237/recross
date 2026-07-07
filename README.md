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

- **Deposits** — `FundsDeposited` events are enumerated from every EVM
  SpokePool via public RPCs. SpokePool addresses are resolved live from the
  HubPool registry (`crossChainContracts`) on Ethereum — nothing hardcoded to
  go stale. Both event generations (`V3FundsDeposited` and the post-migration
  bytes32 `FundsDeposited`) are recognized. The scan window is located by block
  timestamp search, not a fixed blocks-per-hour estimate.
- **Outcomes come from the status API, never from missing logs** — a sample of
  the most-recent settled deposits is classified directly by the
  `deposit/status` API: `filled`/`slowFilled` = success, `expired`/`refunded` =
  failed. This is the key correctness property: a successful fill the scanner
  didn't happen to observe on-chain can **never** be miscounted as a failure,
  because failure is only ever asserted by the API, not inferred from absence.
  (An earlier version matched fill logs and treated "no fill seen" as failure;
  destination-chain fills fell outside the per-chain block window on fast
  chains, producing a wildly inflated ~19% rate.)
- **What's excluded from the rate** — deposits too fresh to have settled
  (younger than a short buffer), deposits the API still reports as in-flight,
  and deposits it can't classify. These are reported separately, not counted.
- **Sampling** — the rate is computed over up to a few hundred of the most
  recent deposits; when the window holds more, the card labels it as a sample.
  Chains whose RPC is unreachable or whose enumeration was truncated are listed
  on the card instead of silently reporting zero. Non-EVM origins (Solana,
  Tron) are not scanned.

## Running

No build step — serve the repo statically and open `index.html`:

```sh
python3 -m http.server
```

Works as-is on any static host (GitHub Pages, Vercel, etc.).
