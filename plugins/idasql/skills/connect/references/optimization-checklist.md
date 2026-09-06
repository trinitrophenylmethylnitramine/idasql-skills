# IDASQL Skills Optimization Checklist

Use this checklist before merging major skill rewrites.

## 1) Routing Clarity
- [ ] `connect` includes explicit intent -> skill routing matrix.
- [ ] Every major skill includes trigger-intent examples in plain user language.
- [ ] Cross-skill handoff instructions are present and unambiguous.

## 2) Anti-Guessing Behavior
- [ ] Schema introspection guidance appears in `connect` and in high-risk skills.
- [ ] Long-tail surfaces reference canonical schema catalog.
- [ ] Decompiler and instruction-heavy queries emphasize required constraints.

## 3) In-Context Learning Strength
- [ ] Every major skill includes warm-start query sequence (`Do This First`).
- [ ] Every major skill includes NL -> SQL examples (not only raw SQL snippets).
- [ ] Examples include interpretation hints (how to read output and decide next step).

## 4) Mutation Safety
- [ ] Mandatory mutation loop is present or referenced.
- [ ] Write examples use precise keys (`func_addr`, `addr`, `idx`, `slot`, etc.).
- [ ] Verify/refresh steps exist after mutation examples.

## 5) Performance and Failure Handling
- [ ] Skills include timeout/empty-result fallback patterns.
- [ ] Skills document high-cost query constraints and safe alternatives.
- [ ] Raw dirtree examples include `tree = ?` and prefer pushed-down `path`/`parent_path` filters; normal organization examples prefer `funcs.folder_path` / `types.folder_path`.
- [ ] HTTP/REPL/CLI startup patterns remain deterministic.

## 6) Big-Database Behavior
- [ ] `connect` carries the Big Database Contract (threshold, server-first, discovery order, session pragma block, liveness/restart).
- [ ] No skill teaches per-query CLI spawns (`idasql -q`) as an iterative workflow on big databases.
- [ ] Every example reading a `pushdown`/`guarded`/`expensive` surface (per `connect/references/schema-catalog.md` cost classes) carries its constraint or `LIMIT`.
- [ ] No example runs unfiltered `pseudocode`/`ctree*` (decompiles every function) or unfiltered `xrefs`/`disasm_calls` aggregates without a small-database caveat.
- [ ] Graph traversals (`call_graph`, `shortest_path`, recursive CTEs) bound depth and `LIMIT` output; depth guidance is present.
- [ ] Output budget guidance exists (column lists, `LIMIT`, ~500-row/~32 KB ceiling, big pulls to files).
- [ ] Deep-dive skills (`re-source`) include the operating loop with resume protocol, campaign ledger keys, checkpoint cadence, and context-pressure protocol.
- [ ] `storage` documents the standard `campaign:*` ledger keys and the session-start read rule.
- [ ] Monster-function guidance present where decompilation is taught (length-sanity check before reading).

## 7) Consistency
- [ ] Terminology is consistent across skills (`addr`, `func_addr`, `start_addr`, etc.).
- [ ] Table/view names match live SQL metadata.
- [ ] Cross-links between related skills are accurate.
