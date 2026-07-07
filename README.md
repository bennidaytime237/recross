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

A completed transfer is **only** `status=success`. Anything else that a request
old enough to have settled ends up in is a failure — the swap/bridge did not
deliver funds.

- **Failures** — every non-`success` request from `api.relay.link/requests/v2`:
  the Relay-confirmed `failure` and `refund` states, plus requests still stuck
  in `pending`/`depositing`/`submitted`/`waiting` after the settle buffer
  (deposited but never delivered and never refunded). Each status is paginated
  over the window via `continuation`; a status the API won't accept as a filter
  is skipped from the headline count (the rate still counts it). If the page cap
  is hit the count is shown as `N+`.
- **Rate** — non-success over all settled requests. The sample only includes
  requests created before `now − settle buffer`, so it isn't skewed by fresh
  requests that fast-settling successes dominate while slower refunds are still
  pending. If the window has more requests than the sample cap, the rate is
  computed over the most recent settled sample and labeled as such.
- **Settle buffer** — requests younger than a few minutes are excluded from
  both the count and the rate; they may still legitimately be in flight.

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
