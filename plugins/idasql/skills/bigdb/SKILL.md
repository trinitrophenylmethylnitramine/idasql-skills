---
name: bigdb
description: "Work efficiently with very large IDA databases (tens of thousands of functions or more). Use when queries feel slow, when opening the database is expensive, when results would flood context, or for deep multi-session investigations on big binaries like il2cpp/game engines."
metadata:
  argument-hint: "[database or question]"
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

---

## Trigger Intents

Use this skill when:
- the database is **big**: more than ~20,000 functions, ~100,000 names, ~500,000 strings, or a >200 MB `.i64` (read the counts from `binary` — it is free)
- a query is slow, times out, or returns partial rows (`timed_out: true`)
- a result set would be large enough to flood the conversation (hundreds of rows, many decompilations)
- the investigation will span many turns or sessions ("understand X deeply", "map the Y subsystem")
- the user names a large target class: il2cpp / Unity games, statically linked C++, firmware images, engines

This skill is a **methodology layer**. It does not replace `connect`, `xrefs`, `decompiler`, or `re-source` — it changes *how* you use them when scale matters. Session mechanics (starting/verifying servers) live in `connect`; the recursive annotation loop lives in `re-source`.

---

## Why Scale Changes Everything

On a reference database (libil2cpp, 452,396 functions, 650,307 names, 578,920 strings) the same queries an agent writes on a 500-function binary behave completely differently:

| Operation | Small DB | 452k-func reference DB |
|---|---|---|
| `idasql -s db.i64 -q "..."` (one process per query) | ~1 s | **~36 s** — database open dominates |
| First `funcs`/`names` read of a session | instant | ~2.8 s once (session cache build) |
| Warm `funcs`/`names` scan, `ORDER BY`, subquery | instant | **~5–30 ms** (idasql ≥0.0.20 session cache; was ~2.7 s per statement) |
| After a write statement | — | next funcs/names read rebuilds (~3 s) — batch writes, read after |
| `grep WHERE pattern = '...'` | instant | ~7 s (iterator scan each time — budget it) |
| `strings WHERE content LIKE '%x%'` | instant | ~0.2 s (cheap — string list is cached in the IDB) |
| `xrefs WHERE to_addr = <EA>` | instant | instant (pushdown works) |
| `xrefs WHERE to_addr IN (literals)` | full scan | **~0 ms** (SQLite loops the O(1) pushdown per value) |
| `xrefs` non-pushdown aggregate (`is_code=1`, joins) | fast | ~10 s per query → **~0.5 s** after `PRAGMA idasql.xrefs_shared_cache = 1` (opt-in; one ~10 s materialization + RAM ~1.5 GB at 24M xrefs) |
| `decompile(ea)` | 50–200 ms | ~0 ms warm, 50–200 ms cold |
| `SELECT COUNT(*) FROM disasm_calls` (full scan) | fast | **>60 s → times out, partial rows** |
| Unfiltered `pseudocode` / `ctree*` | slow | **idasql ≥0.0.19: refused instantly by the `decomp_scan_max_funcs` guard; older: decompiles all 452k functions — hours** |

Numbers were measured with idasql v0.0.18.1; costs scale roughly with function count. Re-measure with the battery in [references/bigdb-patterns.md](references/bigdb-patterns.md) when the tool or database changes.

Three conclusions follow, and they are the whole skill:

1. **Sessions, not processes.** One long-lived server per database; never one CLI process per question.
2. **Pushdown or nothing.** Every expensive table is touched only through its constrained fast path.
3. **Budgets, not hopes.** Every query states its own bound; big results go to files, never into the conversation.

---

## Contract 1 — Session Discipline

- **One long-lived HTTP session per database** (`idasql -s <db> --http <port> -w`), verified with `GET /status` before first use and restarted if it dies. Full startup and recovery procedure: `connect` skill's Big Database Contract.
- **Discovery (idasql ≥0.0.19):** a running CLI server maintains `<db>.idasql-session.json` next to the database (`{port, pid, db_path, started_at, tool_version}`) — read it first; a missing file means no live session (or an unclean exit — confirm via `/status`). `/status` also reports counts, `uptime_s`, `queries_served`, `last_query_ms`.
- **Never answer iterative questions with `idasql -q`** on a big database — each invocation repays the full open cost (tens of seconds to minutes). idasql ≥0.0.19 prints this hint itself on >100k-function databases.
- Queries execute **serially** on the server (decompiler thread affinity). Do not fire concurrent requests at one server; queue them in one script when order matters.
- PRAGMAs are per-session state — a restarted server needs the session pragma block re-run (see `connect`).
- Writes persist only via `SELECT save_database()` or server `-w` + clean `/shutdown`. Saves are expensive at this scale: batch writes, save at checkpoints.

## Contract 2 — Query Budgets

Every query you issue must state its own bound. Before sending, check:

1. **Explicit column list** — never `SELECT *` on `funcs`, `names`, `strings`, `instructions`, `xrefs`, `types`.
2. **`LIMIT` on every row-returning statement** over a high-cardinality surface. Aggregates (`COUNT`, `GROUP BY`) need no LIMIT but must target a constrained surface.
3. **Aggregate first, fetch second.** Answer "which functions…" with `GROUP BY` + `ORDER BY ... LIMIT 20`, then pull details for the shortlist only.
4. **Response ceiling: ~500 rows or ~32 KB into the conversation.** Anything larger goes to a file (`?format=csv` + `curl -o`), then inspect the file with targeted reads/grep. On idasql ≥0.0.19 enforce it server-side too: `PRAGMA idasql.max_rows = 500;` — truncated statements come back `partial:true` with a warning naming the true count.
5. **At most ~3 full decompilations live in context at once.** Extract the facts you need (names, calls, offsets, constants), write them down (annotations / ledger), then move on.
6. **Fan-out bound: ≤20 rows per hop, depth ≤3** for `call_graph` / `shortest_path` / recursive CTEs. In a graph with hundreds of thousands of nodes, depth 5+ reaches nearly everything and proves nothing.

## Contract 3 — Working-Set Discipline

Never explore a big database by browsing. Navigate anchor → neighborhood → shortlist → dig:

1. **ANCHOR** — find candidate EAs using only cheap surfaces: `grep` (name patterns, ~seconds), `strings` + xrefs to strings (~0.2 s per LIKE), `imports`, filtered `disasm_calls WHERE callee_name LIKE ...`. Output: a list of ≤20 function EAs.
2. **EXPAND** — one bounded hop at a time: `xrefs WHERE to_addr = <EA>` (callers, instant), `disasm_calls WHERE func_addr = <EA>` (callees), `call_graph` with small `max_depth`. Keep the fan-out bound; aggregate (`COUNT`) before listing.
3. **SHORTLIST** — rank the expanded set (by size, call count, string hits) and pick the few functions that actually need reading.
4. **DIG** — `SELECT decompile(ea)` for the shortlist only, under Contract 2's three-decompilation rule.

The funcs tax is history on idasql ≥0.0.20: the first unfiltered `funcs`/`names` read of a session builds a session cache (~3 s once at 452k functions); warm scans, `ORDER BY`s, and subqueries then run in milliseconds. Two residual rules: **a write statement drops the cache** (the next read rebuilds — batch your writes instead of interleaving), and wide column reads (`name`, `prototype` for every row) still pay per-row IDA lookups on first touch, so keep projections narrow when you can.

## Contract 4 — Aggregation-First Analysis

Pattern questions ("does the code do X?") do not require reading code:

- Calls/loops/branches/returns per function → `ctree_v_*` views **filtered by `func_addr`** (one decompilation each, streamed).
- Call-graph shape → `disasm_v_leaf_funcs`, `disasm_v_call_chains` (bounded), `call_graph`.
- Dangerous-API usage → `disasm_calls WHERE callee_name IN (...)` + `GROUP BY func_addr`.
- Full `decompile()` is for the final targets a human would actually read.

## Monster-Function Caveats

Big binaries contain huge compiler-generated functions (hundreds of KB). Before decompiling one, length-check it:

```sql
SELECT name, size, length(decompile(addr)) FROM funcs WHERE addr = <EA>;
```

A 397,744-byte function returning 753 chars of pseudocode means the decompiler bailed or truncated — do not analyze the short text as if it were the function. For monsters, sample instead: `instructions WHERE func_addr = <EA>` windows, `blocks`/`cfg_edges` shape, or ctree aggregates.

## Target-Class Notes (il2cpp / Unity and similar)

- Most of the function count is compiler-generated template noise (`RuntimeInvoker_*`, method wrappers, `MethodInfo` plumbing). **Do not browse `funcs` and do not try to "understand the binary" function by function.**
- Anchor on: exported il2cpp entry points, `global-metadata` strings, string references in `strings` (cheap), vtable/type structures, and the minority of genuinely named functions.
- Give up breadth early: define the question, find its 20 anchors, be done.

For worked offload recipes (batch decompile via `dump_pseudocode`, offline export via `--export-sqlite`, result-to-file, IDAPython batching with a persistent sandbox, subagent fan-out, the measurement battery): [references/bigdb-patterns.md](references/bigdb-patterns.md).

---

## Failure and Recovery

- **Query refused: "unfiltered scan refused … would decompile every one of them":** the decompiler guard (idasql ≥0.0.19, `PRAGMA idasql.decomp_scan_max_funcs`, default 20000) stopped a catastrophic full scan. This is the guard working — add the `func_addr` constraint the error suggests. Only disable it (`= 0`) for a deliberate, bounded, you-know-why full pass.
- **Query timed out (`timed_out: true`, partial rows):** the numbers you got are truncated — do not trust aggregates from a timed-out statement. Narrow the scope (pushdown, LIMIT, smaller range) instead of retrying the same scan.
- **Statement came back `partial:true` with a `max_rows` truncation warning:** the cap fired — paginate with `LIMIT/OFFSET` or pull to a file; the warning names the true row count.
- **Server died mid-session:** restart per `connect`'s Big Database Contract, re-run the session pragma block, re-verify anchors (unsaved writes are lost — this is why checkpoints matter).
- **Response too big to reason about:** you broke Contract 2 — pull to a file, summarize from the file, and record the lesson in the ledger.
- **"It's slow":** check which contract was broken. In practice it is almost always per-query CLI spawns (Contract 1) or a missing pushdown (Contract 3).

---

## See Also

- `connect` — Big Database Contract: server startup, liveness, discovery, session pragmas
- `re-source` — the recursive deep-dive loop (RESUME → FRAME → ... → CHECKPOINT) and the campaign ledger
- `storage` — `netnode_kv` ledger key conventions
- `decompiler` — decompiler table constraints and cost model
- `xrefs` — pushdown fast paths and `call_graph` / `shortest_path` parameters
