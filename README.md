# xrpl-router

A request router for serving XRP Ledger history from stock `xrpld` nodes.

The router sits in front of two kinds of `xrpld` node:

- **Live nodes**: normal peer-to-peer `xrpld`, keeping recent history. They serve the newest ledgers,
  requests that name no ledger, and `submit`, `subscribe` and path finding.
- **Range nodes**: `xrpld` started with `--archive` on a fixed, already-imported range of ledgers.
  Each range node serves one slice of history.

Clients see one JSON-RPC and WebSocket endpoint with the normal `xrpld` API.

## What the router does

| Request | Handling |
|---|---|
| Ledger-scoped methods (`account_info`, `book_offers`, `amm_info`, `account_lines`, …) | Read `method` and `ledger_index` / `ledger_hash`, send to the live nodes or the range node holding that ledger, pass the body through |
| `"validated"`, `"current"`, `"closed"`, or no ledger | Live nodes; `"validated"` is rewritten to the explicit ledger number the router knows, so a client never sees an older ledger from a lagging node |
| `ledger_hash` | Resolve the hash to a ledger number, then route |
| `ping` | Answered by the router |
| `server_info`, `server_state`, `ledger_current`, `ledger_closed`, validated `ledger` header | Fetched once per ledger close and served from memory; `fee` with a ~1 s TTL |
| `tx` by hash | Sent to the ranges; the first answer that is not `txnNotFound` wins |
| `account_tx` | Walks ranges newest to oldest (or oldest to newest with `forward`), fills `limit`, merges results, issues its own `marker` |
| `subscribe` | One upstream subscription per live node, fanned out to clients |

## Cache

In process memory, per router. Entries live for one ledger close (~3.5 s), so there is no shared
cache: a restart refills within seconds.

## Status

Design only; no code yet.
