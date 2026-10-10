# PR: fix(web): paginate /contexts after merging all context types

## What's happening

`/contexts` passes the global page offset into each ChromaDB collection
query *before* merging results. With two 10-item collections and `offset=15`
(page 2, limit=15), both collections return nothing — the page is empty
while five records remain.

The root cause is in `read_contexts` (`opencontext/server/routes/web.py`):
when no type filter is set, `get_all_processed_contexts` is called with the
raw `offset`, which the ChromaDB backend applies independently to every
collection it iterates over.

## Fix

When no type filter is active, the route now fetches all records (up to
10 000), merges and sorts them, then slices with the page offset in Python.
Single-type browsing is unchanged — it still passes offset directly to the
backend, which works correctly for a single collection.

## Scope

One-file change confined to `opencontext/server/routes/web.py`. No storage
API changes needed.

## How to verify

1. Create 10 processed contexts of type A and 10 of type B.
2. `GET /contexts?limit=15` — 15 items, `has_next=true`.
3. `GET /contexts?page=2&limit=15` — 5 items, `has_next=false`. (Was empty before.)
4. `GET /contexts?type=A&page=2&limit=8` — single-type path, unchanged.

Fixes #377
