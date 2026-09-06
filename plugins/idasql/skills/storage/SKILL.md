---
name: storage
description: "Persistent key-value storage in IDA databases. Use when asked to store metadata, track progress, or persist session state via netnode_kv."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

---

## netnode_kv

Persistent key-value store backed by IDA netnodes. Data is saved inside the IDB automatically. Supports full CRUD and O(1) key lookup via `WHERE key = '...'`.

| Column | Type | Writable | Description |
|--------|------|----------|-------------|
| `key` | TEXT | — | Unique key (identity, read-only) |
| `value` | TEXT | Yes | Arbitrary-length value (blob storage) |

```sql
-- Store a value
INSERT OR REPLACE INTO netnode_kv(key, value) VALUES('author', 'alice');

-- Read by key (O(1) lookup)
SELECT value FROM netnode_kv WHERE key = 'author';

-- List all entries
SELECT * FROM netnode_kv;

-- Update a value
UPDATE netnode_kv SET value = '2.0' WHERE key = 'version';

-- Delete an entry
DELETE FROM netnode_kv WHERE key = 'author';
```

### Use Cases

- **Session state**: Track analysis progress across sessions (e.g., which functions have been annotated)
- **Metadata**: Store custom metadata like analyst name, analysis date, notes
- **Bookkeeping**: Track which functions have been re-sourced, annotated, or reviewed
- **Configuration**: Store per-database analysis settings

**Session-start rule:** the first data query of any resumed investigation reads the
campaign ledger (below). Progress that isn't read at startup effectively doesn't
exist.

```sql
-- Track analysis progress with ONE KEY PER ITEM (O(1) reads/writes, no rewrite races).
-- Do not store one giant JSON list under a single key: it must be rewritten whole
-- on every update and degrades badly as the campaign grows.
INSERT OR REPLACE INTO netnode_kv(key, value) VALUES('annotated:main', 'done');

-- Read progress in a new session
SELECT value FROM netnode_kv WHERE key = 'annotated:main';
```

---

## Campaign Ledger (standard convention)

Deep investigations (see `re-source`) use two standard key families as their
working memory. Treat them as the workspace-wide convention — any session (human,
agent, or script) can resume a campaign by reading them:

| Key | Value | Rules |
|-----|-------|-------|
| `campaign:ledger` | JSON: `{goal, state, phase, anchors: ["0x…"], open_questions: [], decisions: [], next: [], updated_at}` | One per campaign; rewrite whole; keep <8 KB |
| `campaign:func:<hex>` | JSON: `{status: todo\|doing\|done\|blocked\|skipped, summary, confidence, updated_at}` | O(1) per key; enumerate functions from `campaign:ledger`.anchors, never by `LIKE`-scanning keys |

```sql
-- Resume: first query of a session
SELECT value FROM netnode_kv WHERE key = 'campaign:ledger';

-- Record a finding
INSERT OR REPLACE INTO netnode_kv(key, value) VALUES('campaign:func:401000', json_object(
    'status', 'done',
    'summary', 'process_context: consumes MY_CONTEXT, frees buffer on refcount 0',
    'confidence', 'high',
    'updated_at', datetime('now')));
```

Legacy `re_source:0x…` keys from older campaigns remain valid — same role as
`campaign:func:<hex>`.

---

## Performance Rules

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| `WHERE key = '...'` | O(1) | IDA's netnode `hashval_long()` — always use exact key lookup |
| `WHERE key LIKE 'prefix%'` | O(n) | Scans all entries; acceptable for small datasets |
| `SELECT * FROM netnode_kv` | O(n) | Full netnode scan; fine for typical use (dozens to hundreds of entries) |

**Key rules:**
- Exact key lookup (`WHERE key = '...'`) is O(1) — this is the preferred access pattern.
- Prefix scans (`LIKE 'prefix%'`) iterate all entries but are fast for typical netnode sizes. At campaign scale (thousands of keys) avoid them — keep the anchor list in `campaign:ledger` and look keys up directly.
- netnode_kv is stored inside the IDB file — it persists automatically with `save_database()`.

---

## Advanced Storage Patterns

### JSON-based progress tracking

Store structured analysis state as JSON for richer querying:

```sql
-- Store progress with structured metadata
INSERT OR REPLACE INTO netnode_kv(key, value)
VALUES('progress:overview', json_object(
    'total_funcs', (SELECT COUNT(*) FROM funcs),
    'named_funcs', (SELECT COUNT(*) FROM funcs WHERE name NOT LIKE 'sub_%'),
    'timestamp', datetime('now')
));

-- Read and parse progress
SELECT json_extract(value, '$.total_funcs') AS total,
       json_extract(value, '$.named_funcs') AS named,
       json_extract(value, '$.timestamp') AS ts
FROM netnode_kv WHERE key = 'progress:overview';
```

### Per-function annotation status tracking

Track which functions have been annotated and what was done — one key per function
(prefer the standard `campaign:func:<hex>` family above):

```sql
-- Mark a function as annotated
INSERT OR REPLACE INTO netnode_kv(key, value)
VALUES('campaign:func:401000',
       json_object('status', 'done', 'summary', 'DriverEntry init',
                    'analyst', 'alice', 'date', date('now')));

-- Find unannotated functions by joining with funcs
SELECT f.name, printf('0x%X', f.addr) AS addr
FROM funcs f
WHERE f.name NOT LIKE 'sub_%'
  AND NOT EXISTS (
    SELECT 1 FROM netnode_kv
    WHERE key = 'campaign:func:' || printf('%X', f.addr)
  )
ORDER BY f.size DESC
LIMIT 20;
```

### Naming conventions for keys

Use a `namespace:entity:id` format for organized storage:

```
campaign:ledger             → the active campaign's goal/state/anchors/next (one key)
campaign:func:401000        → per-function status (standard re-source convention)
re_source:0x401000          → per-function status (legacy form, same role)
config:string_minlen        → analysis configuration
snapshot:2024-01-15          → point-in-time analysis snapshot
tag:crypto:401000           → function tags/categories
```

```sql
-- List all keys in a namespace
SELECT key, value FROM netnode_kv WHERE key LIKE 'tag:crypto:%';

-- Count entries per namespace
SELECT SUBSTR(key, 1, INSTR(key, ':') - 1) AS namespace,
       COUNT(*) AS entries
FROM netnode_kv
WHERE key LIKE '%:%'
GROUP BY namespace
ORDER BY entries DESC;
```

---

## See Also

- `annotations` — for analyst breadcrumbs that should live in the address space (comments, bookmarks). Use `netnode_kv` for cross-address session state.
