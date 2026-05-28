# Retrieval Architecture — Evolution and Findings

> This journal documents how the retrieval architecture evolved from its starting point to its current state.
> It is a history of findings, failures, and adaptations — not a spec. For current specs, see `architecture/specifications/`.

---

## Where We Started

### The original problem

Standard RAG fails on enterprise repositories. The failure is not a tuning problem — it is architectural.

Enterprise codebases have properties that break naive retrieval:
- A single business operation spans seven layers (ASPX → JS → ASMX → BLL → WCF → DAL → SQL). No single file contains a complete feature.
- Files are too large. A DAL class with 1,840 lines and 14 methods does not fit in a prompt.
- Embeddings lose architectural relationships. A cosine search for "label printing" returns every file that mentions the word, not the chain of files that implement it.
- Context windows overflow. Sending the raw repo to an LLM is not retrieval — it is brute force.
- Repeated reasoning is expensive. The same architectural questions are answered over and over across sessions.

The key early insight: **enterprise codebases already contain hidden intelligence** — in naming conventions, service chain patterns, folder structures, and DAL relationships. The retrieval system should exploit these deterministic patterns first, before spending any LLM budget.

### Initial design decisions

The first architecture was shaped by three bets:

1. **Regex-based C# extraction** — parse method signatures, class names, and call sites using pattern matching
2. **Flat context files** — store everything in four aggregate files: `workflow-context.json`, `service-context.json`, `dal-context.json`, `ui-context.json` per project
3. **Stack** — Langroid for orchestration, Roo Code for IDE validation, Gemma4:4b as the small model

The proposed retrieval flow:
```
Question → Workflow Detection → Convention Mapping → Context Retrieval
→ Confidence Evaluation → Detailed Expansion → Raw Code Expansion
```

The "Context Registry" in early diagrams was a placeholder for what would become the `.ai/` output layer.

---

## Finding 1 — Regex Extraction Fails on Real Enterprise Code

**What we found:** Running regex-based extraction on the iPAS PreWeigh codebase produced systematically wrong results:

- **Phantom methods from commented-out code.** Regex cannot distinguish `// GetLabelData(orderId)` from an actual method declaration. MoldingBLL appeared to have 218 methods; the real count was 212. Six were commented-out legacy methods.
- **Non-public SVC methods missed.** WCF service files contain non-public `[OperationContract]` implementations. Regex patterns targeting `public` keywords silently skipped them. iPAS_LabelPrintSVC appeared to have 217 methods; the real count after fixing was 347 — a 37% miss rate.
- **Fully-qualified `[WebMethod]` attributes skipped.** Some services use `[System.Web.Services.WebMethod]` instead of `[WebMethod]`. Regex matching only the short form caused entire ASMX services to appear method-free. iPAS_SOPWebService went from 0 detected WebMethods to 229 after the fix.
- **Namespace-qualified ServiceClient calls missed.** BLL files that reference WCF services via `iPAS_MoldSVC.MoldingServiceClient` instead of bare `MoldingServiceClient` were invisible to the call extractor. ManufactureBLL appeared to call 479 SVC methods; the real number after namespace matching was 603.

**Adaptation:** Replaced regex extraction with **tree-sitter AST parsing**. The AST distinguishes comments from code structurally — no heuristics needed. Non-public methods, fully-qualified attributes, and namespace-qualified call sites are all first-class AST nodes.

---

## Finding 2 — Flat Output Files Collide in Multi-Project Solutions

**What we found:** The four-aggregate-file design (`workflow-context.json`, `service-context.json`, etc.) works for a single project but breaks the moment a solution has two projects with a shared DAL class. Both projects try to write `dal-context.json` and overwrite each other's records.

Enterprise ASP.NET solutions routinely contain 10–50 projects. A flat output design cannot survive this.

**Adaptation:** Replaced flat output with a **mirror-path layout** under `.ai/`. Each indexed source file gets its own context file at `.ai/{ProjectName}/{mirrored/path}/{filename}.relationships.json`. No two files in the entire enterprise fleet can collide regardless of how many projects share a filename.

The four aggregate files (`_execution_graph.json`, `_indexes.json`, `_shared_components.json`, `_workflows.json`) live at the `.ai/` root and cover all projects — they are designed for cross-project queries.

---

## Finding 3 — Workflow Needs to Be a First-Class Retrieval Unit

**What we found:** Even with accurate per-file records, the original retrieval design still answered file-level questions. "Show me how MoldingSchedule works" returned the ASPX file, or the ASMX file — not the chain. The developer had to manually assemble the picture across seven layers.

The real unit of reasoning for an enterprise developer is the **workflow** — one atomic business operation from page to SQL table. The retrieval unit should match that.

**Adaptation:** Built `_workflows.json` as a first-class output artifact.

- One entry per ASMX WebMethod (= one atomic business operation)
- Each entry captures the full chain: `pages → webmethod → bll_methods → svc_methods → dal_methods → dal_classes → sql_tables`
- Workflows are grouped by `entity_base` (derived by stripping CRUD verb prefixes and noise suffixes from the WebMethod name) — so all CRUD operations on `MoldingSOPTemplate` form one retrievable cluster

**Verified scale on iPAS PreWeigh:**
- 1,970 workflow entries across 16 ASMX services
- 1,804 entity groups
- 922 full-chain (depth 4+ — page through to SQL table)
- 82 no-downstream (utility/enum endpoints — expected, not failures)
- 850 BLL-only (CommonBLL infrastructure or external compiled DLLs — surfaced honestly)

This file is now the #1 differentiator of the platform. A developer asking about a feature gets the entire chain in one deterministic lookup — no LLM, no vector search, no guessing.

---

## Finding 4 — Stack Pivots During Phase 1

Two stack decisions changed during the build:

**Langroid → LangFlow.** Langroid was the initial orchestration choice. During Phase 1 implementation it became clear that LangFlow's visual pipeline builder was a better fit for a platform that will need rapid experimentation across LLM providers and retrieval strategies. LangFlow also has native support for the tool-calling patterns the Phase 2 enrichment pipeline requires. Langroid was replaced throughout the stack.

**Roo Code → Continue.** Roo Code was the initial IDE validation tool. Continue proved to be a better fit for the IDE layer — it has a more mature plugin model, better support for custom context providers (which is how the MCP server will surface inline), and does not require cloud connectivity. All IDE validation references were updated to Continue.

**Gemma4:4b removed.** This small model was listed as a secondary inference option. After evaluating actual throughput on enrichment tasks, it was removed from the active stack. Qwen3:8b (reasoning) and Qwen2.5-Coder:1.5b (autocomplete) cover the use cases it was intended for, with better results.

---

## Finding 5 — MCP Server Is the Right Interface for All Callers

**What we found:** As the Phase 1 artifacts matured, it became clear that every caller — LangFlow pipelines, Claude Desktop, the Continue IDE plugin, custom scripts — was going to need the same queries: "what does this file do?", "who calls this method?", "what breaks if I change this table?". Without a standard interface, each caller would have to re-implement its own JSON parsing logic.

**Adaptation:** Designed the **MCP Server** (Phase 5) as a stateless protocol adapter. The server exposes the `.ai/` JSON files as six callable tools over the Model Context Protocol. Every caller gets the same interface regardless of how they are built. The server is deliberately thin — no LLM calls, no caching, no cross-project joins. Intelligence lives in the JSON files, not in the server.

---

## Where We Reached (May 2026)

### Architecture

```
Source code
    ↓ (tree-sitter AST parser)
Per-file .relationships.json  +  Aggregate files (_execution_graph, _indexes, _workflows, ...)
    ↓ (MCP Server — Phase 5 design complete)
Unified tool interface for all callers
    ↓ (Phase 2 LLM enrichment — not yet started)
Semantic layer (rh, br, dt, lc, qa, cfg) added to each .relationships.json
    ↓ (Phase 4 Qdrant — deferred)
Vector fallback over enriched retrieval hints
```

### Validated numbers

| Metric | Value |
|---|---|
| ASPX pages indexed | 237 |
| ASMX services | 16 |
| WebMethods | 1,970 |
| WCF services | 16 |
| BLL classes | 8 |
| DAL classes | 16 |
| SQL tables | 606 |
| Methods (~total) | ~9,600 |
| Execution graph nodes | 12,413 |
| Execution graph edges | 25,922 |
| BLL→SVC call accuracy | 99.8% |
| Scan rate | ~3,000 files/min |
| Workflow entries | 1,970 |
| Workflow groups | 1,804 |
| Full-chain workflows (depth 4+) | 922 |

### Stack

| Component | Current |
|---|---|
| Orchestration | LangFlow |
| IDE | Continue |
| Local inference | Ollama |
| Reasoning | Qwen3:8b |
| Autocomplete | Qwen2.5-Coder:1.5b |
| Embeddings (Phase 4) | nomic-embed-text |
| Vector store (Phase 4) | Qdrant |
| AST parsing | tree-sitter |

### What changed vs. where we started

| Area | Started with | Reached |
|---|---|---|
| Extraction method | Regex pattern matching | tree-sitter AST (structurally correct) |
| Output layout | Flat 4-file per project | Mirror-path `.ai/` (collision-free at scale) |
| Retrieval unit | File or method | Workflow (full chain, entity-grouped) |
| Orchestration | Langroid | LangFlow |
| IDE | Roo Code | Continue |
| Small model | Gemma4:4b | Qwen2.5-Coder:1.5b |
| Caller interface | Direct JSON reads | MCP Server (Phase 5 design complete) |

---

## Key Principle — Unchanged Throughout

**Raw code is the last resort.**

Every design decision — per-file context records, workflow entries, graph traversal, MCP tools, retrieval ordering — was made to push raw code reads as far back as possible. The agent should be able to answer 90%+ of developer questions using the structured `.ai/` artifacts alone, without opening a single source file.

This principle survived every pivot.
