# bigdb — Worked Patterns

Recipes referenced by the `bigdb` SKILL.md. All examples assume a long-lived HTTP
session (see `connect`) at `http://127.0.0.1:8199` — substitute your port.

---

## 1. Measurement Battery

Run this once per database (and again after tool upgrades) to calibrate every budget
decision. Expected shape on the 452k-function reference DB is given after each step.

```bash
# a) CLI open cost — one query, timed (this is what per-query CLI spawns pay every time)
time idasql -s <db>.i64 -q "SELECT COUNT(*) FROM funcs"
# reference: ~36 s total, count = 452396

# b) With the HTTP session up:
q() { curl -s -m 300 -X POST http://127.0.0.1:8199/query --data-binary "$1"; }

q "SELECT key, value FROM binary
    WHERE key IN ('funcs_count','names_count','strings_count','segments_count');"
# reference: instant; 452396 / 650307 / 578920 / 17

q "SELECT COUNT(*) FROM funcs;"                 # reference: ~2.7 s (full scan)
q "SELECT COUNT(*) FROM names;"                 # reference: ~2.8 s
q "SELECT COUNT(*) FROM grep WHERE pattern='Player%';"   # reference: ~7 s
q "SELECT COUNT(*) FROM strings WHERE content LIKE '%http%';"  # reference: ~0.2 s
q "SELECT COUNT(*) FROM xrefs WHERE to_addr = <literal EA>;"   # reference: ~0 ms
q "SELECT length(decompile(<literal EA>));"     # reference: ~0 ms warm, 50-200 ms cold
q "SELECT COUNT(*) FROM ctree_lvars WHERE func_addr = <literal EA>;"  # ~3 ms
```

Do **not** run the negative examples (unfiltered `pseudocode`/`ctree*`, `COUNT(*) FROM
disasm_calls` or `xrefs`) — they are the failure modes this skill exists to prevent
(hours of decompilation; 60 s timeout with partial rows).

If a statement returns `timed_out: true`, its numbers are partial — narrow, don't retry.

---

## 2. Big Pull → File → Inspect

When a result legitimately needs more than ~500 rows / ~32 KB (exports, full string
tables, call lists for offline correlation), send it to a file and inspect selectively:

```bash
# CSV to file (agent context stays clean)
curl -s -X POST 'http://127.0.0.1:8199/query?format=csv' \
  --data-binary "SELECT addr, name, size FROM funcs WHERE name LIKE 'il2cpp_%' LIMIT 5000" \
  -o anchors.csv

# Then work the file, not the conversation:
#   Grep for patterns, Read slices, or hand the file path to a subagent.
```

Other server-side file producers: `gen_listing(path)` (whole-database listing — huge,
generate rarely), `gen_cfg_dot_file(addr, path)` (per-function CFG).

---

## 3. Batch Decompile via dump_pseudocode (preferred, idasql >= 0.0.20)

`dump_pseudocode(path, target)` decompiles server-side and writes files — pseudocode
never passes through the response. `target` is either an integer EA (dump that one
function to exactly `path`) or a funcs-folder path (one `<EA>.c` per function in the
folder + `manifest.json`; returns a summary).

```sql
-- Single function to an exact file
SELECT dump_pseudocode('E:/rev/dump/entry.c', 0xB7FDDC);
-- -> {"addr":"0xB7FDDC","name":"nullsub_7479","ok":true,"bytes":81,"path":"..."}

-- Batch: curate a folder, then dump it
INSERT INTO dirtree_folders(tree, path) VALUES('funcs', 'idasql/dump');
UPDATE funcs SET folder_path = 'idasql/dump'
WHERE addr IN (0x123456, 0x1234A0 /*, ... */);
SELECT dump_pseudocode('E:/rev/dump/batch', 'idasql/dump');
-- -> {"ok":true,"dumped":2,"failed":0,"manifest":"E:/rev/dump/batch/manifest.json",...}
```

Then inspect the dumped files selectively (Grep for the pattern, Read the hits) —
never the whole set. Folder dumps honor `PRAGMA idasql.decomp_scan_max_funcs` and
the query-interrupt flag, so a runaway dump is refused/cut short.

## 4. Offline Export (--export-sqlite, idasql >= 0.0.20)

Materialize the structural surfaces once, then explore with plain sqlite3 — no IDA
involved, no open cost, unlimited re-querying. Measured on the 452k-function
reference DB (one-time cost):

```bash
idasql -s bigdb.i64 --export-sqlite offline.db
#   funcs: 36 s | names: 9 s | strings: 0.5 s | disasm_calls: 103 s | xrefs: 48 s
#   total ~3m50s -> 452k funcs, 650k names, 579k strings, 5.8M calls, 24.2M xrefs
```

```bash
sqlite3 offline.db "SELECT f.name, COUNT(*) c FROM xrefs x JOIN funcs f ON f.addr=x.to_addr GROUP BY x.to_addr ORDER BY c DESC LIMIT 10;"
```

Reference timings offline: whole-program 24M-row hotspot aggregate ~6.4 s (repeatable,
no IDA); add indexes once (`CREATE INDEX ix ON xrefs(to_addr)`, ~11 s) and targeted
queries drop to milliseconds. Workflow: **materialize → explore offline → return to
live IDA only for decompile + annotate.** The exported file is a snapshot — re-export
after big annotation campaigns if you need names/folders fresh.

## 5. IDAPython Custom Passes (when SQL can't express it)

`dump_pseudocode` covers plain batch decompilation. Use IDAPython only for custom
per-function analysis (correlation, clustering, custom visitors). Enable once per
session:

```sql
PRAGMA idasql.enable_idapython = 1;
PRAGMA idasql.idapython_output_max = 20000;  -- hard cap on returned print output
```

```sql
SELECT idapython_snippet('
import ida_hexrays, ida_funcs, ida_lines, json, os
os.makedirs(r"E:\rev\dump", exist_ok=True)
targets = [0x123456, 0x1234A0]  # shortlist EAs from the working-set step
manifest = []
for ea in targets:
    f = ida_funcs.get_func(ea)
    if not f:
        manifest.append({"addr": hex(ea), "ok": False, "error": "no func"}); continue
    cf = ida_hexrays.decompile_func(f)
    if not cf:
        manifest.append({"addr": hex(ea), "name": ida_funcs.get_func_name(ea),
                         "ok": False, "error": "decompile failed"}); continue
    text = "\n".join(ida_lines.tag_remove(l.line) for l in cf.get_pseudocode())
    path = r"E:\rev\dump\%X.c" % f.start_ea
    open(path, "w", encoding="utf-8").write(text)
    manifest.append({"addr": hex(f.start_ea), "name": ida_funcs.get_func_name(ea),
                     "ok": True, "bytes": len(text), "path": path})
open(r"E:\rev\dump\manifest.json", "w").write(json.dumps(manifest, indent=1))
print("dumped:", sum(1 for m in manifest if m["ok"]), "of", len(manifest))
', 'campaign');
```

Rules of the road:

- **One sandbox key per campaign** (`'campaign'` above): globals persist across calls
  with the same key, so you can keep state (processed lists, caches) without resending.
- Return a **manifest + one summary line**, never the pseudocode itself. Read the
  dumped files selectively (Grep for the pattern, Read the hits).
- Length-sanity-check each result (`bytes` in the manifest) against function size —
  see Monster-Function Caveats in SKILL.md.
- Write durable findings back through SQL (`funcs.rpt_comment`, `netnode_kv`) so they
  survive outside the sandbox.

---

## 6. Subagent Fan-Out

When your harness supports parallel subagents, use them as **bounded question
executors**, not explorers. The parent owns the session, the ledger, and all writes.

Parent pattern per subagent task:

```
Question: which of these functions parses the protocol header?
Session: http://127.0.0.1:8199 (already running — do NOT start another server;
         queries are serialized; issue your queries sequentially)
Addresses (exact EAs): 0x..., 0x..., 0x...   (≤10)
Budget: ≤20 queries, ≤500 rows each; decompile at most these functions
Return: written findings only — one JSON or markdown block:
        {addr, verdict, evidence (2-3 lines), confidence}. No raw dumps.
Do not: rename, comment, or write anything. Report only.
```

Collect results, then the parent applies annotations and ledger updates in one batch.
This keeps each context small and the IDB writes serialized and reviewable.

---

## 7. Working-Set Walk-Through (il2cpp-style target)

Goal: "find the save-game encryption."

```sql
-- ANCHOR: cheap surfaces only
SELECT addr, content FROM strings WHERE content LIKE '%save%game%' LIMIT 25;
SELECT addr, content FROM strings WHERE content LIKE '%crypt%' OR content LIKE '%aes%' LIMIT 25;
-- reference cost: ~0.2 s each

-- Who references the promising strings? (point lookups are instant)
SELECT from_addr, type FROM xrefs WHERE to_addr = <string EA>;

-- Containing functions for the ref sites (bounded, one hop)
SELECT addr, name, size FROM funcs
WHERE addr IN (SELECT from_func FROM xrefs WHERE to_addr = <string EA>)
LIMIT 20;

-- EXPAND one hop, bounded: callees of the top candidate
SELECT callee_addr, callee_name FROM disasm_calls
WHERE func_addr = <candidate EA> LIMIT 20;
SELECT COUNT(*) FROM xrefs WHERE to_addr = <candidate EA>;   -- fan-in signal

-- SHORTLIST: rank by size/strings/callers, pick ≤3
-- DIG: decompile the shortlist, extract constants (key schedule tables?), record
```

Then: annotate the findings, write `campaign:func:<hex>` ledger entries, checkpoint
(`SELECT save_database()`). Continue with `re-source` for the recursive part.
