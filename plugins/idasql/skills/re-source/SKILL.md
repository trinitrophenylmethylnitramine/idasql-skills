---
name: re-source
description: "Re-source IDA binaries. Use when asked for recursive annotation, structure recovery, type reconstruction, or bottom-up program understanding."
metadata:
  argument-hint: "[function-name or address]"
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

For structure recovery patterns, see: `references/struct-recovery-patterns.md`

This skill teaches a **methodology** for recovering source-level understanding from compiled binaries using idasql. It orchestrates the other skills into a systematic workflow.

Deep re-sourcing outlives a single context window. The methodology is therefore
**resumable by construction**: every valuable fact is written into the IDB
(annotations + campaign ledger) before the conversation moves on, and every session
starts by resuming from that state instead of from memory.

---

## The Operating Loop

| # | Phase | Purpose | Surfaces | Budget |
|---|-------|---------|----------|--------|
| 0 | **RESUME** | Re-enter an existing campaign | `netnode_kv` ledger | ≤3 queries |
| 1 | **FRAME** | One question + evidence plan | ledger write | 1 write |
| 2 | **ANCHOR** | Cheap candidate discovery | `grep`, `strings`+`xrefs`, `imports`, filtered `disasm_calls` | ≤20 anchors |
| 3 | **EXPAND** | Bounded neighborhood | `xrefs` point lookups, `disasm_calls WHERE func_addr=`, `call_graph` (depth ≤3) | ≤20/hop |
| 4 | **DIG** | Read the shortlist | `decompile(ea)`, ctree | ≤3 in context |
| 5 | **ANNOTATE** | Make findings durable | `annotations`/`types` writes, `funcs.rpt_comment`, folders | verify each |
| 6 | **LEDGER** | Record progress + next steps | `netnode_kv` | 1 write/step |
| 7 | **CHECKPOINT** | Persist everything | `SELECT save_database()` | every ~10 funcs / ~30 min |

### 0. RESUME — always the first move

```sql
SELECT value FROM netnode_kv WHERE key = 'campaign:ledger';
```

If a campaign exists, continue it: re-derive anchor EAs from the ledger's
`anchors` list (not from memory of earlier turns), check per-function status with
O(1) key lookups, and proceed to the `next` items. If not, start at FRAME.

### 1. FRAME

State, and record in the ledger: one question, its success criteria, and the
evidence plan (which surfaces, rough query budget). A session without a framed
question drifts — that drift is what "getting lost" looks like from the inside.

### 2. ANCHOR — discovery on cheap surfaces only

Pick candidates via `grep` (name patterns), `strings` + xrefs to interesting
strings, `imports`, or `disasm_calls WHERE callee_name LIKE ...`. On a big database
follow the `bigdb` skill's contracts — never browse `funcs` as a discovery method.
Output: a shortlist of ≤20 function EAs, recorded in the ledger.

### 3. EXPAND — bounded neighborhood

One hop at a time, with fan-out caps. In dense graphs (100k+ functions),
`max_depth` 5+ reaches nearly everything and proves nothing — keep depth ≤3 and
`LIMIT` every traversal. Details and SQL: see **Expand: callees and callers** below.

### 4. DIG — read the shortlist

`SELECT decompile(ea)` for the few functions that made the cut. Extract facts
(names, calls, offsets, constants), then write them down (ANNOTATE) before pulling
the next decompilation. Keep ≤3 full decompilations in context at once. Details:
see **Dig: read, then annotate** below.

### 5. ANNOTATE — the IDB is long-term memory

Every stabilized fact becomes an annotation: rename, prototype, lvar type,
pseudocode comment, and a one-line `funcs.rpt_comment` summary (this is what makes
the function findable by later queries). Move reviewed functions into a campaign
folder (`UPDATE funcs SET folder_path = 'idasql/<campaign>/reviewed' WHERE addr = ...`).
Follow the `annotations` skill's Function Summary contract and Mandatory Mutation
Loop (read → edit → refresh → verify).

### 6. LEDGER — see below

### 7. CHECKPOINT

```sql
SELECT save_database();
```

Saves are expensive on big databases — checkpoint at natural boundaries (every
~10 functions or ~30 minutes, and always before ending a session), not after every
write.

---

## Campaign Ledger (netnode_kv)

Standard keys (see `storage` for the full netnode_kv reference):

| Key | Value | Rules |
|-----|-------|-------|
| `campaign:ledger` | JSON: `{goal, state, phase, anchors: ["0x…"], open_questions: [], decisions: [], next: [], updated_at}` | One per campaign; rewrite whole; keep <8 KB |
| `campaign:func:<hex>` | JSON: `{status, summary, confidence, updated_at}` with `status ∈ todo\|doing\|done\|blocked\|skipped` | O(1) per key |

```sql
-- FRAME: create/update the campaign ledger
INSERT OR REPLACE INTO netnode_kv(key, value) VALUES('campaign:ledger', json_object(
    'goal', 'Recover the protocol context struct and its lifecycle',
    'state', 'active', 'phase', 'expand',
    'anchors', json_array('0x401000','0x401050'),
    'open_questions', json_array('who frees ctx->buffer?'),
    'decisions', json_array('MY_CONTEXT is refcounted, not owned'),
    'next', json_array('decompile 0x401050 callers'),
    'updated_at', datetime('now')));

-- LEDGER: record a function finding (O(1) key)
INSERT OR REPLACE INTO netnode_kv(key, value) VALUES('campaign:func:401000', json_object(
    'status', 'done',
    'summary', 'process_context: consumes MY_CONTEXT, frees buffer on refcount 0',
    'confidence', 'high',
    'updated_at', datetime('now')));

-- RESUME: read per-function status by key (do not LIKE-scan)
SELECT value FROM netnode_kv WHERE key = 'campaign:func:401000';
```

Cautions:

- Enumerate functions from `campaign:ledger`.anchors — never `key LIKE 'campaign:%'`
  scans (O(n) over all netnode entries; fine for dozens, toxic at thousands).
- Legacy `re_source:0x…` keys from older campaigns remain valid; map them mentally
  to `campaign:func:<hex>`.
- The ledger records **progress and decisions**. The durable technical knowledge
  belongs in annotations (`rpt_comment`, types, names) — the ledger points at it.

---

## Context-Pressure Protocol

When outputs start flooding (a turn blew past its budget, or you can feel the
session nearing its limits):

1. Stop pulling new data. Finish interpreting what you have.
2. ANNOTATE what you learned (renames/comments/types).
3. LEDGER: update `campaign:ledger` (`state`, `phase`, `next`) so the next session
   resumes exactly here.
4. Drop the raw text from your reasoning; keep only written-down facts.
5. Continue with the next FRAME, from a clean slate.

A lost context with a current ledger costs minutes. A lost context without one
costs the whole investigation.

---

## Definition of Done (per function)

A function is `done` when it has:
- a meaningful **name**,
- a best-known **prototype**,
- a one-line **`rpt_comment` summary**,
- **types** applied where recoverable (args, locals, struct fields),
- a **folder** assignment (`idasql/<campaign>/reviewed` or finer).

Until then it stays `doing`. Batch sizing: one annotation pass = ≤10 functions;
report progress as ledger counts (`done/doing/todo`), not by re-listing functions
in prose.

---

## Phase Details

### Dig: read, then annotate

Start from an anchor or resumed ledger target:

```sql
-- Decompile the target function
SELECT decompile(0x401000);

-- Or by name
SELECT decompile('DriverEntry');
```

Use the `annotations` skill to edit the decompilation into something readable:

```sql
-- Rename local variables to meaningful names
UPDATE ctree_lvars SET name = 'driver_object' WHERE func_addr = 0x401000 AND idx = 0;
UPDATE ctree_lvars SET name = 'registry_path' WHERE func_addr = 0x401000 AND idx = 1;

-- Apply types to arguments/locals
UPDATE ctree_lvars SET type = 'PDRIVER_OBJECT'
WHERE func_addr = 0x401000 AND idx = 0;

-- Inspect pseudocode anchors before writing comments
SELECT line_num, addr, line, comment
FROM pseudocode
WHERE func_addr = 0x401000
ORDER BY line_num;

-- Add inline comments explaining logic
-- Example below uses a previously resolved writable anchor, not the function entry row.
UPDATE pseudocode SET comment = 'initialize dispatch table'
WHERE func_addr = 0x401000 AND addr = 0x401020;

-- Verify changes
SELECT decompile(0x401000, 1);
```

Set the function summary (makes the function indexable for later queries; exact
trigger semantics follow the `annotations` skill's Function Summary contract):

```sql
SELECT addr, name, comment, rpt_comment
FROM funcs
WHERE addr = 0x401000;

UPDATE funcs
SET rpt_comment = 'DriverEntry: initializes driver dispatch routines and device object'
WHERE addr = 0x401000;
```

### Expand: callees and callers (bounded)

Follow calls inside the function, annotate each callee, building understanding
bottom-up — one hop at a time, fan-out ≤20:

```sql
-- List callees to visit (bounded)
SELECT callee_name, printf('0x%X', callee_addr) as addr
FROM disasm_calls WHERE func_addr = 0x401000
LIMIT 20;

-- Or map the call subtree (keep max_depth small; in dense graphs depth 5
-- reaches nearly everything and proves nothing)
SELECT func_name, depth FROM call_graph
WHERE start = 0x401000 AND direction = 'down' AND max_depth = 3
LIMIT 50;
```

Follow callers to build the bigger picture: how is this function used?

```sql
-- Who calls this function?
SELECT caller_name, printf('0x%X', caller_addr) as addr
FROM callers WHERE func_addr = 0x401000
LIMIT 20;

-- Transitive callers (bounded)
SELECT func_name, depth FROM call_graph
WHERE start = 0x401000 AND direction = 'up' AND max_depth = 3
LIMIT 50;

-- Shortest path from an entry point (max_depth is a search bound, not a
-- result size — always LIMIT the output)
SELECT step, func_name FROM shortest_path
WHERE from_addr = (SELECT addr FROM funcs WHERE name = 'main')
  AND to_addr = 0x401000 AND max_depth = 10;
```

On big databases, prefer the recursion through point lookups over graph TVFs when
you need context at each step (the CTE pattern under Advanced Patterns), and
consult `bigdb` for budgets.

### Structure Recovery

The hardest part. Decompiled code often shows casts like `*(DWORD *)(a1 + 0x10)` — these are structure field accesses.

#### Step-by-Step Process

**a) Identify offset patterns in a single function:**
```sql
-- Look at the decompiled code for cast patterns
SELECT decompile(0x401000);

-- Query ctree for pointer arithmetic (field accesses)
SELECT addr, op_name, num_value
FROM ctree WHERE func_addr = 0x401000
  AND op_name IN ('cot_add', 'cot_idx')
  AND num_value IS NOT NULL;
```

**b) Cross-function correlation — find more fields:**
```sql
-- Find all callers that pass the same struct pointer
SELECT DISTINCT (SELECT name FROM funcs WHERE dc.func_addr >= addr AND dc.func_addr < end_addr LIMIT 1) as caller
FROM disasm_calls dc
WHERE dc.callee_addr = 0x401000;

-- Decompile each caller and look for more offset accesses
-- Each caller may reveal different fields of the same struct
```

**c) Callee correlation — let callees reveal field types:**
```sql
-- What does the function call with the struct pointer?
SELECT COALESCE(call_obj_name, call_helper_name) AS callee_name,
       arg_idx, arg_op, arg_num_value
FROM ctree_call_args
WHERE func_addr = 0x401000
  AND arg_var_name = 'a1';
-- If callee expects HANDLE, the field at that offset is a HANDLE
```

**d) Build the struct incrementally:**
```sql
-- Create the struct
INSERT INTO types (name, kind) VALUES ('MY_CONTEXT', 'struct');

-- Get the ordinal
SELECT ordinal FROM types WHERE name = 'MY_CONTEXT';

-- Add fields as you discover them (ordinal derived from the name above)
INSERT INTO types_members (type_ordinal, member_name, member_type)
SELECT ordinal, 'handle', 'void *' FROM types WHERE name = 'MY_CONTEXT';
INSERT INTO types_members (type_ordinal, member_name, member_type)
SELECT ordinal, 'buffer_ptr', 'void *' FROM types WHERE name = 'MY_CONTEXT';
INSERT INTO types_members (type_ordinal, member_name, member_type)
SELECT ordinal, 'buffer_size', 'unsigned int' FROM types WHERE name = 'MY_CONTEXT';
```

**e) Apply the recovered struct:**
```sql
-- Make this step self-contained (MY_CONTEXT was recovered in step d above)
SELECT parse_decls('struct MY_CONTEXT { void *handle; void *buffer_ptr; unsigned int buffer_size; };');

-- Apply to function prototype
UPDATE funcs SET prototype = 'int __fastcall process_context(MY_CONTEXT *ctx);'
WHERE addr = 0x401000;

-- Or apply to a local variable
UPDATE ctree_lvars SET type = 'MY_CONTEXT *'
WHERE func_addr = 0x401000 AND idx = 0;

-- Re-decompile to verify clean rendering
SELECT decompile(0x401000, 1);
```

Recovering a struct across many functions on a big database? Pull the caller set to
a file or one IDAPython batch pass (`bigdb` offload patterns) instead of
decompiling them one query at a time.

---

## Advanced Re-Sourcing Patterns (CTEs)

### Transitive caller discovery

Find all functions that transitively pass a struct through a chain of calls — who ultimately provides the data?

> **Prefer `call_graph` for simple traversal:** `SELECT func_name, depth FROM call_graph WHERE start = 0x401000 AND direction = 'up' AND max_depth = 3 LIMIT 50` replaces the CTE below. Use the CTE only when you need to JOIN caller context (e.g. offset accesses) at each step.

```sql
-- Recursive CTE: walk callers up to 5 levels
WITH RECURSIVE caller_chain AS (
    -- Base: direct callers of the struct-consuming function
    SELECT c.caller_func_addr AS func_addr,
           c.caller_name AS func_name,
           1 AS depth
    FROM callers c
    WHERE c.func_addr = 0x401000

    UNION ALL

    -- Recurse: callers of callers
    SELECT c.caller_func_addr,
           c.caller_name,
           cc.depth + 1
    FROM caller_chain cc
    JOIN callers c ON c.func_addr = cc.func_addr
    WHERE cc.depth < 5
)
SELECT DISTINCT func_name, printf('0x%X', func_addr) AS addr, MIN(depth) AS min_depth
FROM caller_chain
GROUP BY func_addr
ORDER BY min_depth;
```

### Aggregate offset accesses across all callers

Build a comprehensive struct field map by collecting offset patterns from every function that touches the struct:

```sql
-- Collect field offset accesses from all functions that call process_context
WITH callers_of AS (
    SELECT DISTINCT func_addr
    FROM disasm_calls
    WHERE callee_addr = 0x401000
),
offset_accesses AS (
    SELECT func_addr,
           (SELECT name FROM funcs WHERE func_addr >= addr AND func_addr < end_addr LIMIT 1) AS func_name,
           num_value AS field_offset,
           op_name
    FROM ctree
    WHERE func_addr IN (SELECT func_addr FROM callers_of)
      AND op_name IN ('cot_add', 'cot_idx')
      AND num_value IS NOT NULL
      AND num_value BETWEEN 0 AND 0x1000
)
SELECT field_offset,
       printf('0x%X', field_offset) AS hex_offset,
       COUNT(DISTINCT func_addr) AS seen_in_funcs,
       GROUP_CONCAT(DISTINCT func_name) AS functions
FROM offset_accesses
GROUP BY field_offset
ORDER BY field_offset;
```

### Find struct-heavy functions (candidates for structure recovery)

Functions with the most `cot_add` offset patterns are likely manipulating structs through raw pointer arithmetic:

```sql
-- Functions with most pointer arithmetic (struct field access candidates)
-- Note: the ctree IN-list decompiles each seed function once; keep the seed
-- set small (this is a per-function cost, not a table scan).
WITH offset_funcs AS (
    SELECT func_addr,
           COUNT(*) AS offset_accesses,
           COUNT(DISTINCT num_value) AS unique_offsets
    FROM ctree
    WHERE func_addr IN (SELECT addr FROM funcs ORDER BY size DESC LIMIT 100)
      AND op_name = 'cot_add'
      AND num_value IS NOT NULL
      AND num_value BETWEEN 1 AND 0x1000
    GROUP BY func_addr
)
SELECT (SELECT name FROM funcs WHERE func_addr >= addr AND func_addr < end_addr LIMIT 1) AS name,
       printf('0x%X', func_addr) AS addr,
       offset_accesses,
       unique_offsets
FROM offset_funcs
ORDER BY unique_offsets DESC
LIMIT 15;
```

### Cross-reference struct field offsets with known type sizes

Match observed offsets against field sizes of existing types to guess field types:

```sql
-- Compare observed offsets with known struct sizes
WITH observed AS (
    SELECT DISTINCT num_value AS offset
    FROM ctree
    WHERE func_addr = 0x401000
      AND op_name = 'cot_add'
      AND num_value IS NOT NULL
      AND num_value BETWEEN 0 AND 0x200
),
candidate_types AS (
    SELECT t.name AS type_name, t.size AS type_size
    FROM types t
    WHERE t.is_struct = 1 AND t.size > 0
)
SELECT o.offset, printf('0x%X', o.offset) AS hex_offset,
       ct.type_name, ct.type_size
FROM observed o
LEFT JOIN candidate_types ct ON ct.type_size = o.offset
ORDER BY o.offset;
```

---

## Key Principles

1. **Bottom-up understanding**: Start with leaf callees, annotate them, then work up to callers. Each annotated callee makes the caller easier to understand.

2. **Cross-function struct correlation**: A single function rarely reveals the full struct layout. Look at multiple callers/callees of the same function to discover different fields.

3. **Iterative refinement**: Apply what you know, re-decompile, see if the output improves. Add more fields/types as you discover them.

4. **Verify every edit**: Follow the Mandatory Mutation Loop (read → edit → refresh → verify) from the `annotations` skill.

5. **Write before you drop**: annotations + ledger before dropping raw output from context — the IDB and the ledger are the campaign's memory, the conversation is scratch.

6. **Checkpoint on a cadence**: `SELECT save_database()` every ~10 functions or ~30 minutes, and always before ending a session.

7. **Bounded expansion**: fan-out ≤20 per hop, graph depth ≤3; aggregate before listing.

---

## Related Skills

- **`bigdb`** — working at scale: session discipline, query budgets, offload patterns (batch decompile to files, subagent fan-out)
- **`annotations`** — The editing/annotation workflow: how to rename, retype, comment
- **`decompiler`** — Deep decompiler reference: ctree, types, parse_decls, union selection
- **`types`** — Type system mechanics: struct/union/enum creation and manipulation
- **`xrefs`** — Caller/callee traversal, `call_graph` / `shortest_path` tables, `string_refs` view
- **`disassembly`** — `cfg_edges` for control flow understanding during struct recovery
- **`storage`** — netnode_kv reference and the campaign ledger key conventions
