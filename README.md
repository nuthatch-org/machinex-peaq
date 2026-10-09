# machinex-peaq

A [nuthatch](https://github.com/nuthatch-org/nuthatch) nest: **MachineX (MX-V3) on peaq**.

A Solidly/Ramses-style DEX: concentrated-liquidity and legacy pools discovered from their factories,
gauges and fee distributors from the Voter, and position lifecycle from the position manager.

One binary, one config file, no graph-node, no gateway, no query fees.

> **Status: building, not available.** A tip-following run works and decodes correctly, but the
> factory children cannot be discovered on any public peaq endpoint. Read *Read this before trusting
> it* before you rely on this for anything.

## Why this exists

The published subgraph (`EAVSLJ9r1mc18RmDXFHUxzH1QiQ83hDx1MY8LcPQ3nBB`, deployment
`QmcssauLtPam7J4pmAbvCKVoaBH6mowXhQJpLRMEyoQsEL`) was reported twelve days behind chain head on
2026-08-26. The cause was an indexer-side node stall rather than a defect in the subgraph, so unlike
`pancakeswap-infinity-cl-bsc` this nest is a second source rather than a rescue.

Scaffolded from that deployment CID:

```sh
nuthatch init --from-subgraph QmcssauLtPam7J4pmAbvCKVoaBH6mowXhQJpLRMEyoQsEL \
  --chain peaq --rpc https://evm.peaq.network
```

## What it indexes

**Chain:** `peaq` (chain id 3338, not in nuthatch's built-in registry - the config carries its own
endpoints and takes the unregistered-chain finality policy). **5 contracts**, **5 templates**,
**7 factory rules**, **40 declared tables** plus one `__children` table per template.

| alias | address | start block |
|---|---|---|
| `cl_factory` | `0x2646dcbe025d21a2925fdaceb639e998e17d6060` | 5,248,275 |
| `fee_collector` | `0x8dc13a3fc0a5a533dee8a00f8c68589e640b000e` | 5,248,275 |
| `voter` | `0x3af1dd7a2755201f8e2d6dcda1a61d9f54838f4f` | 5,248,275 |
| `legacy_factory` | `0xa3f356f0403b4f10345cd95e0c80483fddd63ebd` | 5,248,275 |
| `nonfungible_position_manager` | `0x4dbc7dd463bb4a8c6c4381cceeb58b9327c7cfa4` | 5,248,275 |

## Verified

Indexed blocks **11,341,876 to 11,361,878** and stored **99 rows** across 4 tables: 45 position
fee collections, 22 liquidity increases, 17 position transfers and 15 liquidity decreases. Decode is
correct on everything that fired. **Every other table was empty** (4 of 44 in the live nest had rows), for the reasons below - that is
the honest result of this run, not a summary of what the nest could do with a better endpoint.

## Read this before trusting it

- **The Voter contract has no code on peaq.** `eth_getCode` on
  `0x3aF1dD7A2755201F8e2D6dCDA1a61d9f54838f4f` returns `0x` at `latest` on all three public endpoints
  tested (`evm.peaq.network`, `peaq-rpc.publicnode.com`, `peaq.api.onfinality.io`). The address comes
  straight from the subgraph manifest, so `GaugeCreated`, `CustomGaugeCreated`, `GaugeKilled`,
  `GaugeRevived` and `Whitelisted` can never have fired, and the four gauge factory rules and the fee
  distributor rule can never discover a child. The rules are kept because they are correct if the
  address is ever right; the tables will stay empty until it is.
- **No public peaq endpoint is archive, and none of them says so.** `eth_getCode` on the CL factory
  returns 6,298 bytes at `latest` and `0x` at block 5,248,285. Wide `eth_getLogs` at deployment height
  returns `[]` rather than an error. A backfill from 5,248,275 therefore completes clean with nothing
  in it, which is the worst way to be wrong. Measured 2026-08-26 across all three endpoints.
- **Consequence: pool and gauge tables need archive access to ever populate.** Children are discovered
  from `PoolCreated` / `PairCreated`, which fired near 5,248,275. A tip-window run never sees them, so
  every `cl_pool_template__*`, `legacy_pool_template__*` and `*_gauge_template__*` table stays empty.
  Point `--rpc` at an archive peaq node and this nest becomes what it is meant to be.
- **The factory rules are hand-written, not inferred.** `init --from-subgraph` reports candidates but
  cannot resolve which template a creation event spawns, because that decision lives in the mapping
  WASM. Both gauge templates are wired to both Voter creation events deliberately: the two ABIs share
  event *names* but no signatures - `ClaimRewards` is `(uint256,bytes32,address,address,uint256)` on
  the CL gauge and `(address,address,uint256)` on the legacy one - so topic0 routes each log into
  exactly one table and watching an address under both templates cannot double-count.
- **Not a port of the subgraph's entities.** Decoded event tables only. Anything the mapping computed
  with a contract call or a USD pricing graph is not here.

## Run it

```sh
nuthatch init --from https://github.com/nuthatch-org/machinex-peaq
cd machinex-peaq
nuthatch dev --dir . --backfill 20000 --window 2000
nuthatch sql --dir . "SELECT count(*) FROM nonfungible_position_manager__collect"
```

For anything past a tip window, pass your own archive endpoint with `--rpc` and check it first with
`nuthatch doctor --rpc <url>`. Note that `--window` is not honoured by the factory discovery pass,
which issues wide windows that `evm.peaq.network` and onfinality answer with
`"query timeout of 10 seconds exceeded"` after 10s; `peaq-rpc.publicnode.com` answers with an explicit
`"exceed maximum block range: 50000"`, which nuthatch does recognise and split on.

## Tables

40 declared tables across 5 contracts and 5 templates, plus one `__children` table per template. See `schema.json`, or `nuthatch sql --dir . ".tables"`.

## Licence

`MIT OR Apache-2.0`, same as nuthatch.
