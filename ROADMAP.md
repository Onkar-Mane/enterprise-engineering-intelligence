# Platform Roadmap

Central phase plan for the Enterprise Engineering Intelligence Platform.

**Reference codebase (iPAS PreWeigh):** 237 ASPX pages · 16 ASMX services · 1,970 WebMethods · 16 WCF · 8 BLL · 16 DAL · 606 SQL tables · ~9,600 methods

---

## Phase 1 — Deterministic Scanner ✅ DONE

**Goal:** Build a fully deterministic, LLM-free scanner that maps the entire codebase structure into structured JSON artifacts.

**What was built:**
- tree-sitter AST parser for C#, JavaScript, ASPX
- Per-file `.relationships.json` under `.ai/` mirror-path layout
- `_execution_graph.json` — 12,413 nodes, 25,922 edges
- `_indexes.json` — exact lookup tables (page→webmethods, webmethod→pages, dalclass→tables, etc.)
- `_shared_components.json` — reusable infrastructure ranked by reference count
- `_phase2_manifest.json` — ordered work list for Phase 2 LLM batching
- `_workflows.json` — **#1 differentiator**: 1,970 workflow entries, 1,804 groups, 922 full-chain (depth 4+)

**Validated accuracy:**
- BLL→SVC call detection: 99.8% (MoldingBLL 212/212, ManufactureBLL 479/480)
- Scan rate: ~3,000 files/min (CPU-only, no LLM)

**Key specs:** `context-generator-spec.md`, `context-file-schema-v1.md`, `workflows-schema.md`, `edge-cases.md`

---

## Phase 2 — LLM Enrichment ⬜ NOT STARTED

**Goal:** Run LLM over each file's `batch_plan` to populate the semantic layer (Layer 2) in every `.relationships.json`.

**What will be built:**
- LangFlow pipeline reading `_phase2_manifest.json`
- LLM writes `rh` (retrieval hints), `br` (brief per-region), `dt` (detailed context), `lc` (local common: magic numbers, shared variables, patterns, gc_candidates)
- `error_symptom_router` — connects runtime error messages to the workflow/file that caused them
- One LLM call per batch (≤5 methods AND ≤1000 lines)

**Estimated throughput:** ~2,000 files/hr (Qwen3:8b via Ollama)

**Prerequisite:** Phase 1 complete ✅

**Key specs:** `context-file-schema-v1.md` (Layer 2 fields), `_phase2_manifest.json` format, `agent-behavior-spec.md`

---

## Phase 3 — Detailed Context ⬜ NOT STARTED

**Goal:** Deep-dive enrichment pass — fill in the `dt` (detailed) layer with method-level conditions, edge cases, magic number explanations, and developer-answered `qa` entries.

**What will be built:**
- Developer Q&A workflow — agent surfaces unanswered questions, developer answers once, stored permanently in `qa`
- `cfg` (conflicts) resolution — developer annotates duplicate/overloaded signatures
- `gc_candidates` promotion — developer reviews and approves entities for `GlobalCommon.json`

**Prerequisite:** Phase 2 complete

---

## Phase 4 — Vector Index (Qdrant) ⬜ NOT STARTED

**Goal:** Build a vector index over Phase 2 semantic fields (`rh`, `br`) to enable natural-language fallback search when deterministic lookup returns no match.

**What will be built:**
- Qdrant index over `rh` and `br` fields (not raw code)
- `search_semantic` MCP tool becomes active
- Embeddings: `nomic-embed-text` via Ollama

**Design decision:** Qdrant is deliberately deferred until Phase 2 fields exist — indexing raw code produces inferior recall compared to indexing LLM-generated retrieval hints.

**Prerequisite:** Phase 2 complete (needs `rh` + `br` fields to index)

---

## Phase 5 — MCP Server 🔵 DESIGN DONE — BUILD NEXT

**Goal:** Expose all Phase 1 artifacts as callable tools over the Model Context Protocol so any agent (LangFlow, Claude Desktop, Continue, custom) can query codebase intelligence with a uniform interface.

**What will be built:**
- Stateless MCP server (reads `.ai/` JSON on every call — no cache)
- 6 tools:
  1. `trace_execution_chain` — walk execution chain upstream/downstream from any node
  2. `find_callers` — reverse lookup: who calls this method?
  3. `find_table_users` — which pages touch this SQL table?
  4. `get_file_context` — return full `.relationships.json` for one file
  5. `list_workflow_files` — return the full chain for one workflow or group
  6. `search_semantic` — vector fallback (active only after Phase 4)
- Query routing: naming/workflow match → graph traversal → semantic → raw code

**Full spec:** `architecture/specifications/mcp-server-spec.md`

**Prerequisite:** Phase 1 complete ✅ (can build against Phase 1 artifacts now; `search_semantic` activates after Phase 4)

---

## Phase 6 — IDE Deep Integration ⏸ DEFERRED

**Goal:** Surface codebase intelligence directly inside the developer's IDE (Continue plugin) with inline context, workflow-aware autocomplete, and impact warnings on edit.

**Deferred until:** MCP server stable and Phase 2 enrichment complete.

---

## Phase 7 — Multi-Solution Fleet ⏸ DEFERRED

**Goal:** Scale to the full enterprise fleet (multiple `.sln` files, cross-solution dependency tracking, shared project deduplication).

**Design:** Covered in `context-generator-spec.md` §8 (Multi-Project Solution Handling).

**Deferred until:** Single-solution pipeline fully validated.

---

## Stack Reference

| Component | Tool |
|---|---|
| Orchestration | LangFlow |
| IDE validation | Continue |
| Local inference | Ollama |
| Reasoning model | Qwen3:8b |
| Autocomplete model | Qwen2.5-Coder:1.5b |
| Embeddings (Phase 4) | nomic-embed-text |
| Vector store (Phase 4) | Qdrant |
| AST parsing | tree-sitter |
