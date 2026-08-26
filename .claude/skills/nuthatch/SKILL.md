---
name: nuthatch
description: Query this self-hosted nuthatch nest on peaq - decoded events, balances, and read-only SQL. Use when asked about on-chain activity for these contracts.
---

# Querying the nuthatch nest

Contracts indexed on peaq:
- `cl_factory` = 0x2646dcbe025d21a2925fdaceb639e998e17d6060
- `fee_collector` = 0x8dc13a3fc0a5a533dee8a00f8c68589e640b000e
- `voter` = 0x3af1dd7a2755201f8e2d6dcda1a61d9f54838f4f
- `legacy_factory` = 0xa3f356f0403b4f10345cd95e0c80483fddd63ebd
- `nonfungible_position_manager` = 0x4dbc7dd463bb4a8c6c4381cceeb58b9327c7cfa4

Data is local - never call an external API for it.

## Preferred: MCP
If a `nuthatch` MCP server is configured, use its tools. Call `schema` first to learn the
data model, then `sql` / `entity` / `balance` / `top_balances`.

## Fallback: HTTP (a `nuthatch dev` must be running)
- Recent rows:  `curl localhost:8288/entities?limit=20`
- Read-only SQL: `curl -G localhost:8288/sql --data-urlencode 'q=SELECT count(*) FROM transfers'`

`sql` sees finalized data only; balances/entity cover the live tip.
