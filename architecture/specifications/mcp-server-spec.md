# MCP Server Specification

> **Status:** Design complete, build next (Phase 5).
> **Purpose:** Expose the Phase 1 deterministic artifacts (`.ai/` JSON files) as callable tools over the Model Context Protocol so any agent — LangFlow, Claude Desktop, Continue, custom — can query the codebase intelligence with the same interface.

---

## 1. Design Principles

1. **Stateless.** Every tool call reads the relevant `.ai/` JSON file on disk. No in-memory cache, no warm-up. Restart anywhere, no state to recover.
2. **Multi-project.** One server instance serves all 20+ projects in the fleet. Every tool call takes a `project=` parameter that maps to `.ai/{project}/`.
3. **Deterministic first.** Default query path is naming/workflow match on `_indexes.json` + `_workflows.json`. Vector search (Qdrant) is only invoked when the deterministic path returns no match — and only in Phase 4+ when Qdrant is built.
4. **JSON is source of truth.** The server never invents data. If a record is missing from the JSON, the server returns "not found" — it does not fall back to LLM inference.
5. **Raw code is the last resort.** Tools return summaries, paths, and line ranges. The caller opens raw source only if the structured response is insufficient.

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Callers: LangFlow / Claude Desktop / Continue / custom     │
└──────────────────────────┬──────────────────────────────────┘
                           │ MCP protocol
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   MCP Server (stateless hub)                │
└──────┬─────────┬─────────┬─────────┬─────────┬──────────────┘
       │         │         │         │         │
       ▼         ▼         ▼         ▼         ▼
  _execution  _indexes  _workflows  *.relat-  _shared_
  _graph      .json     .json       ionships  components
  .json                             .json     .json
                            (per file, per project, under .ai/)
```

The server is a thin protocol adapter. All intelligence lives in the JSON files emitted by the Phase 1 scanner.

---

## 3. Tools

### 3.1 `trace_execution_chain`

**Purpose:** Walk the full execution chain from a starting point in any direction.

**Inputs:**
- `project` (string, required) — project folder name under `.ai/`
- `start` (string, required) — node id (e.g., `page:Mold/ManageMoldingSchedule.aspx`, `webmethod:iPAS_MoldService/ScheduleUnScheduleOrderInfo`, `dal:MoldingDAL.UpdateScheduleInfo`)
- `direction` (enum, default `downstream`) — `downstream` (callees) or `upstream` (callers)
- `max_depth` (int, default `10`) — depth cap to prevent runaway traversals

**Output:** Ordered list of nodes grouped by layer, with the exact edge path from `start` to each.

**Data source:** `_execution_graph.json`

---

### 3.2 `find_callers`

**Purpose:** Reverse lookup — "who calls this method?"

**Inputs:**
- `project` (string, required)
- `target` (string, required) — fully qualified method or DAL class name (e.g., `MoldingDAL.UpdateScheduleInfo`)

**Output:** List of pages, WebMethods, and BLL methods that transitively reach `target`. Each entry carries the graph path.

**Data source:** `_indexes.json` (reverse lookups) + `_execution_graph.json` (path reconstruction)

---

### 3.3 `find_table_users`

**Purpose:** "Which pages read or write this SQL table?"

**Inputs:**
- `project` (string, required)
- `table` (string, required) — table name in canonical form (`LDB1_CAUFV_HDR`, etc.)

**Output:** List of pages that touch the table, with the DAL class and (when known) the specific DAL method.

**Precision note:** Returns are `exact` when per-method SQL extraction is available; otherwise `approximate` (class-level — entire DAL class touches the table, but the specific method may not). The response includes a `precision` flag on every entry. See edge-cases.md limitation #2.

**Data source:** `_indexes.json` (`table_to_pages`, `dalclass_to_tables`)

---

### 3.4 `get_file_context`

**Purpose:** Return the full context record for one source file.

**Inputs:**
- `project` (string, required)
- `rel_path` (string, required) — relative path under the project root (e.g., `UI/Pages/Mold/ManageMoldingSchedule.aspx`)

**Output:** The complete `{rel_path}.relationships.json` for that file. After Phase 2 this includes the semantic fields (`br`, `rh`, `dt`, `lc`, `qa`, `cfg`); before Phase 2 it returns only the deterministic layer.

**Data source:** `.ai/{project}/{rel_path}.relationships.json` (direct read)

---

### 3.5 `list_workflow_files`

**Purpose:** Return the full workflow group — all files that participate in one feature.

**Inputs:**
- `project` (string, required)
- `workflow_or_group` (string, required) — either a workflow id (`iPAS_MoldService/ScheduleUnScheduleOrderInfo`) or a group id (`iPAS_SOPWebService/MoldingSOPTemplate`)

**Output:** The full chain — page(s), WebMethod, BLL methods, SVC methods, DAL methods, DAL classes, SQL tables. For a group, the union across all member workflows.

**Data source:** `_workflows.json`

---

### 3.6 `search_semantic` *(Phase 4+ only — requires Qdrant)*

**Purpose:** Fallback vector search when deterministic lookups return no match.

**Inputs:**
- `project` (string, required)
- `query` (string, required) — natural-language query
- `top_k` (int, default `10`)

**Output:** Ranked list of file context records with similarity scores.

**Data source:** Qdrant index over Phase 2 `rh` and `br` fields (not raw code).

**Status:** Not buildable until Phase 2 (LLM enrichment) and Phase 4 (Qdrant indexing) are complete.

---

## 4. Query Routing Order

Every developer query — whether from LangFlow, Claude Desktop, or an IDE plugin — follows the same routing order:

1. **Naming / workflow match** (free, deterministic). Try `list_workflow_files` and `find_callers` with extracted symbols from the query. If a match is found, return.
2. **Graph traversal** (free, deterministic). If the query references a known node, use `trace_execution_chain` or `find_table_users`.
3. **Semantic search** (Phase 4+ only). Fall through to `search_semantic` if both deterministic paths return empty.
4. **Raw code expansion** (last resort). Only when the structured response is judged insufficient by the caller (LLM-side decision, with explicit user confirmation).

This ordering keeps the common case free and fast, and reserves LLM/vector cost for genuinely ambiguous queries.

---

## 5. Error Handling

| Condition | Response |
|---|---|
| `project` not found under `.ai/` | `404 project_not_found` with list of known projects |
| `rel_path` not found in project | `404 file_not_found` |
| Symbol not found in `_indexes.json` | `404 symbol_not_found` — do **not** fall back to LLM guess |
| `_workflows.json` missing for project | `503 workflows_not_generated` — re-run Phase 1 with `--clean` |
| `search_semantic` called before Phase 4 | `501 not_implemented_in_phase` |

---

## 6. Non-Goals

- **No write operations.** The MCP server is read-only against `.ai/`. Regeneration happens by re-running the Phase 1 scanner.
- **No LLM calls.** The server does not embed an LLM. Callers do their own reasoning over server responses.
- **No caching.** Re-reading JSON on every call keeps the server stateless and avoids stale-cache bugs after a Phase 1 rescan.
- **No cross-project joins in the protocol layer.** Multi-project queries are the caller's responsibility — call the tool once per project.

---

## 7. Open Questions

1. **Authentication.** Local single-developer use needs none. Team/server deployment will need at minimum API-key auth — TBD when team deployment is in scope.
2. **Rate limiting.** Unlikely to be needed given stateless JSON reads; revisit if a caller starts flooding `trace_execution_chain` with deep traversals.
3. **Streaming responses.** Some traversals could return thousands of nodes. MCP supports streaming — to be decided per tool based on real query patterns.
