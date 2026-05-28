# Competitive Landscape: Enterprise Engineering Intelligence

> **Purpose:** This document maps every major AI code intelligence tool against the problem space that this project addresses. It explains where existing tools fall short for large enterprise codebases, and clarifies which design decisions in this project are differentiated vs. which are incremental improvements.

> **Implementation status (May 2026):** Phase 1 (deterministic scanner) is complete — tree-sitter AST extraction, per-file `.relationships.json`, execution graph (12,413 nodes / 25,922 edges), indexes, shared components, and **`_workflows.json`** (1,970 workflow entries, 1,804 groups, 922 full-chain) are all production-ready. **The workflow-unit-as-retrieval differentiator is now emitted, not just claimed.** Phase 2 (LLM enrichment: `br`/`rh`/`dt` fields) has not yet started. Phase 5 (MCP server) is design-complete and build-next. AST parity with structural-graph tools (Cody/SCIP, CodeGraph) is achieved for the .NET WebForms layer stack.

---

## The Core Problem (Restated)

Standard AI coding assistants are built and benchmarked on modern, greenfield codebases — typically TypeScript/Python monorepos with clean layering. Enterprise systems look nothing like this:

```
LabelPrint → ASPX Page → JavaScript → ASMX Web Service → BLL Class →  WCF Service → DAL Class → SQL Stored Procedure
```

A single business workflow spans 7+ layers, potentially 50+ files, hundreds of thousands of lines. When a developer asks "what happens if I change this stored procedure's output format?", an AI tool needs to trace that impact across all layers — deterministically, completely, and fast enough for interactive use.

**None of the existing tools solve this end-to-end.** Here is why, in detail.

---

## Tool-by-Tool Analysis

### 1. Microsoft GraphRAG

**What it is:** A general-purpose RAG framework that builds an LLM-extracted knowledge graph from any text corpus. Published by Microsoft Research.

**Retrieval mechanism:**
- Chunks the input corpus into text segments
- Calls an LLM on every chunk to extract entities and relationships
- Runs Leiden community detection over the resulting graph
- Generates LLM-written summaries for each community
- At query time: Global Search (map-reduce over community summaries) or Local Search (entity neighborhood expansion)

**Why it fails for enterprise code:**

| Problem | Impact |
|---------|--------|
| Not code-aware | Treats function names, class names, and SQL as prose tokens. Cannot reason about call stacks or interface implementations without LLM inference. |
| Non-deterministic indexing | A 2025 study (arXiv 2601.08773) found LLM-extracted knowledge graphs skipped **31.2% of files** during extraction. Enterprise compliance requires 100% coverage. |
| Prohibitive indexing cost | Full indexing of a large enterprise codebase can cost tens of thousands of dollars in LLM API calls. Not viable for internal tooling. |
| No incremental update | Any code change requires re-running the full pipeline. No delta sync. |
| No call graph | Cannot answer "who calls this method?" without LLM inference, which is unreliable. |
| Not designed for code | GraphRAG's own documentation positions it for documents, reports, and unstructured text — not source code. |

**Takeaway:** GraphRAG's community-report approach is powerful for *understanding large text corpora* (e.g., summarizing years of incident reports). It is architecturally wrong for code intelligence where relationships must be precise and exhaustive.

---

### 2. Cursor (Repo-Map + Codebase Search)

**What it is:** The leading AI-first IDE. Uses AST-based chunking + a proprietary embedding model + Turbopuffer (serverless vector DB) for codebase-aware code completion and chat.

**Retrieval mechanism:**
- Tree-sitter parses files into AST-boundary-aligned chunks (~500 tokens each)
- Custom embedding model vectorizes each chunk
- Two-stage retrieval: vector nearest-neighbor search → AI re-ranker
- Incremental sync via Merkle tree file hashing; 5-minute polling interval
- Enterprise optimization: index reuse across team members (92% similarity rate)

**Why it falls short for enterprise scale:**

| Problem | Impact |
|---------|--------|
| File hallucination at scale | Documented **20% error rate on repos >10,000 files** — Cursor edits files that don't exist. |
| Memory exhaustion | Users report 100GB+ RAM usage on large monorepos. |
| 2,500-file local index cap | Multi-service enterprise monorepos routinely exceed this. Remote indexing has higher limits but requires cloud upload. |
| Cloud-only | All embeddings processed server-side. No air-gapped or on-premise option. Not viable for enterprises with strict data residency requirements. |
| No dependency graph | Cursor cannot answer "what is the blast radius of changing this DAL method?" — it finds semantically similar code, not structurally dependent code. |
| No workflow-unit concept | Cursor searches for *relevant snippets*, not *complete business workflows*. A multi-layer query gets fragmented results from each layer independently. |

**Takeaway:** Cursor is excellent for modern greenfield codebases under ~5,000 files where cloud storage of code is acceptable. For regulated industries (finance, healthcare, government) or large legacy codebases, both the scale limits and the cloud dependency are blockers.

---

### 3. Aider (Repo-Map)

**What it is:** An open-source terminal-based AI coding assistant. Uses a deterministic tree-sitter + Personalized PageRank approach to build a compact repo-map — no embeddings, no cloud dependency.

**Retrieval mechanism:**
- Tree-sitter extracts all symbol definitions and references across all files
- Builds a bipartite tag graph: files ↔ symbols, with edges for "defines" and "references"
- Runs Personalized PageRank seeded by files currently in the chat context
- Renders top-ranked symbols as a compact signature map (not full code bodies)
- Default token budget: 1,000 tokens for the map

**Why it falls short for enterprise workflows:**

| Problem | Impact |
|---------|--------|
| User-driven file specification | Aider documentation explicitly states: "Aider currently relies on the user to specify which source files will need to be modified." PageRank surfaces hints, but the user decides what to include. Enterprise workflows have 50+ relevant files — the user cannot enumerate them all. |
| No semantic search | Pure structural analysis. Cannot answer "find all authentication-related code" — only structural relationships. |
| No blast-radius analysis | PageRank ranks *relevance*, not *impact*. It cannot tell you which downstream files break if you change an interface. |
| Token budget vs. enterprise scale | A 1,000-token map across a 100,000-file codebase loses almost all context. Even the expanded mode hits context window limits. |
| No workflow concept | Aider operates file-by-file. There is no concept of a "business workflow" that spans multiple architectural layers. |

**What Aider does well (and this project should learn from):**
- Deterministic, reproducible indexing — same codebase always produces the same map
- Compact signature-based representation instead of full code bodies — keeps token usage low
- Zero-config local operation with full Ollama support

**Takeaway:** Aider's repo-map is the closest philosophical ancestor to the Context File approach. The key difference: Aider's map is computed at query time from a generic graph; this project's Context Files are pre-computed, human-verified, workflow-aware units that exist *persistently* and do not need recomputation on every query.

---

### 4. Continue.dev

**What it is:** An open-source, local-first AI coding assistant and VS Code/JetBrains extension. Uses four parallel indexes: LanceDB embeddings, SQLite code snippets, SQLite full-text search, and raw chunked content.

**Retrieval mechanism:**
- Tree-sitter extracts symbols into `code_snippets` SQLite table
- `transformers.js` runs locally in the extension process to produce embeddings (no data leaves the machine)
- Incremental sync via file hash delta against `tag_catalog`
- Query: embed → LanceDB vector search + BM25 keyword search + symbol lookup → aggregate → LLM

**Why it falls short for enterprise workflows:**

| Problem | Impact |
|---------|--------|
| No reranking | Multi-index results are aggregated but not re-ranked by a secondary model. Result quality is lower than Cursor's two-stage pipeline. |
| 5MB file size cap | Large enterprise files (generated code, minified bundles, large XML configs) are silently skipped. |
| Local embedding model quality | Default `transformers.js` model is weak. Production-quality results require configuring Voyage AI or similar external embedding service. |
| No dependency graph | Same gap as Cursor — semantic similarity, not structural dependency. |
| No workflow unit | `@codebase` search returns relevant snippets; it has no concept of a business workflow that must be retrieved as a complete unit. |

**What Continue.dev does well (and this project should align with):**
- Fully local, air-gapped operation — first-class Ollama support
- Incremental sync is efficient and file-hash-based
- The multi-index approach (embeddings + keyword + symbols) is the right retrieval foundation

**Takeaway:** Continue.dev is the most architecturally aligned tool with this project's goals (local, open, enterprise-capable). The gap is the same as with other tools: no concept of pre-computed workflow-level context units, and semantic search quality degrades at enterprise scale.

---

### 5. Cody by Sourcegraph

**What it is:** An AI coding assistant built on top of the Sourcegraph platform. Uses Sourcegraph's Repo-level Semantic Graph (RSG) built from SCIP (Sourcegraph Code Intelligence Protocol) indices — the most precise code graph among all tools reviewed.

**Retrieval mechanism:**
- SCIP indexers (per language) produce precise symbol graphs: definitions, references, interface implementations, call chains
- RSG enables graph traversal: callers, callees, implementors, cross-repo references
- BM25 keyword search alongside semantic graph traversal
- Pointwise ML ranker scores retrieved context items
- Up to 10 repositories searched simultaneously (Enterprise)

**Why it falls short for enterprise self-hosting:**

| Problem | Impact |
|---------|--------|
| Requires Sourcegraph infrastructure | The precise code graph features only work with a running Sourcegraph instance (self-hosted or managed). Without it, Cody is basic keyword search. |
| SCIP index maintenance burden | SCIP index must be rebuilt on code changes — requires integrating `src-cli` indexer into CI/CD pipeline per language. |
| Code sent to external LLM | Even self-hosted Sourcegraph routes LLM calls to Anthropic/OpenAI externally (~28KB code snippets per request). True air-gap not achieved. |
| Enterprise-only (post-July 2025) | Free and Pro tiers removed. Cost barrier for internal tooling evaluation. |
| No COBOL/RPG/legacy language SCIP | Enterprise legacy systems in COBOL, RPG, PL/SQL, or AS/400 have no SCIP indexer. |
| Operational complexity | Sourcegraph + SCIP indexers + Ollama is a multi-component deployment. Maintenance overhead is high. |

**What Cody/Sourcegraph gets right (this project's north star for graph precision):**
- SCIP-based precise code graph is the gold standard for "what is impacted by this change"
- Cross-repo reference tracking is essential for microservice architectures
- The RSG "expand and refine" traversal pattern (start at a symbol, expand via graph edges, refine by link prediction) is the right algorithmic approach

**Takeaway:** Cody proves that precise code graphs unlock qualitatively better results than embedding-only tools. But the operational complexity and cloud dependency make it unsuitable for enterprise air-gap requirements. This project should adopt the *concept* (pre-computed workflow graphs) while achieving it through lightweight JSON Context Files rather than a full Sourcegraph deployment.

---

### 6. CodeGraph (colbymchenry/codegraph)

**What it is:** An open-source MCP server that builds a deterministic SQLite knowledge graph from tree-sitter ASTs. The most structurally similar tool to this project's approach.

**Retrieval mechanism:**
- Tree-sitter parses all files into ASTs
- 22 node types + 12 edge types covering all structural code relationships
- Stored in SQLite (zero external dependencies)
- OS-native file watchers for real-time incremental sync (2-second debounce)
- Exposes 7 MCP tools to agents: search, context, callers, callees, **impact** (blast-radius), node details, file listing

**Benchmark results (7 large codebases):**
- 35% cost reduction, 59% fewer tokens, 70% fewer tool calls, 49% faster execution vs. no graph

**Why it still falls short for enterprise workflows:**

| Problem | Impact |
|---------|--------|
| No workflow concept | CodeGraph knows about files, functions, and call relationships — but has no concept of a *business workflow* that spans multiple architectural layers. |
| 1MB file size cap | Silently skips large files. |
| No pre-computed summaries | Every query requires graph traversal at runtime. There is no cached, human-verified description of what a component does. |
| No Q&A knowledge capture | No way to record that "this stored procedure must always be called after sp_LockRecord or data corruption occurs." That operational knowledge is lost. |
| No conflict/issue tracking | No way to flag "this component has a known issue with null handling in legacy mode." |
| Framework coverage limited | 13 web frameworks. Does not cover WCF, ASMX, legacy .NET Web Forms, SAP ABAP, or mainframe patterns. |

**What CodeGraph gets right (and this project should match):**
- SQLite-only — zero infrastructure dependencies, embeddable in any tool
- Deterministic, reproducible indexing
- Blast-radius impact analysis via call graph traversal
- The MCP tool interface is the right integration point for modern AI agents

**Takeaway:** CodeGraph is the closest existing tool to this project's intent. The key differentiators here are: (1) the Context File layer that adds human-verified summaries and operational knowledge on top of structural relationships, (2) the workflow-unit abstraction that groups cross-layer components into a single retrievable entity, and (3) the pre-computed hierarchical context that agents read without performing graph traversal at query time.

---

### 7. GitHub Copilot Workspace / Copilot Agent Mode

**What it is:** GitHub's agentic coding product. The agent plans multi-step tasks, executes tool calls (read, search, edit, run terminal) autonomously, and creates pull requests. As of September 2025, this is generally available as "Copilot Coding Agent."

**Retrieval mechanism:**
- GitHub-hosted semantic index (for GitHub-hosted repos)
- Multi-tool search: semantic search, text search, grep, symbol usages, file reading, directory listing
- Agent plans a sequence of tool calls — not a single retrieval step
- Recalculates context on every keystroke (for completion mode)
- MCP support for third-party tool integration

**Why it falls short for enterprise legacy codebases:**

| Problem | Impact |
|---------|--------|
| 2,500-file local index cap | Enterprise monorepos exceed this immediately. |
| 20% file hallucination at >10,000 files | Agent edits non-existent files on large repos. |
| Cloud-only | No air-gapped or on-premise deployment. All code processed on GitHub/Azure infrastructure. |
| No explicit workflow understanding | The agent explores the codebase via tool calls — it does not have pre-loaded knowledge of what a business workflow spans. For a 7-layer enterprise workflow, the agent makes dozens of sequential tool calls to discover what a human developer already knows. |
| Unsigned commits | Pull requests created by the agent have unsigned commits — a compliance problem in regulated industries. |
| Cannot exhaustively analyze | Documentation explicitly acknowledges it "cannot answer 'how many times is this function called across the entire repo'" — a fundamental capability gap for impact analysis. |

**What Copilot Workspace gets right:**
- The *agentic planning* model (multi-step, tool-calling) is the right architecture for complex refactoring
- Combining semantic search + keyword search + symbol tracing is the right multi-modal retrieval approach
- MCP integration means external tools (like this project's Context Files) can be plugged in as first-class tools

**Takeaway:** Copilot Workspace's agent model would be significantly improved if it could read pre-computed Context Files as its first retrieval step, eliminating the dozens of exploratory tool calls it makes to discover what a seasoned enterprise developer already knows. This project's Context Files are *complementary* to agentic tools like Copilot — they are the knowledge base the agent should query first.

---

## The Gap This Project Fills

Across all seven tools, three capabilities are consistently absent:

### Gap 1: Pre-Computed, Human-Verified Context

Every tool reviewed computes context *at query time* from raw source files. For a 100,000-file enterprise codebase, this means either:
- Expensive embeddings over millions of chunks (Cursor, Continue.dev)
- LLM extraction calls over every file (GraphRAG)
- Graph traversal across hundreds of thousands of nodes (CodeGraph, Cody)

None of them store *accumulated knowledge about the codebase* between sessions. If a developer has learned that "StoredProc_X must never be called outside of a transaction," that knowledge is nowhere in the graph — it lives in tribal memory or a comment somewhere.

**Context Files solve this:** They are pre-computed, persistent, human-augmentable knowledge units stored in `.ai-memory/`. Agents read a 400-token context file instead of traversing thousands of nodes.

---

### Gap 2: Workflow-as-Retrieval-Unit

Every tool retrieves *code artifacts* (files, functions, chunks). None retrieve *business workflows* — the cross-layer unit that actually maps to how enterprise developers think.

When a developer asks "how does label printing work?", the correct retrieval unit is not individual functions but the entire LabelPrint → ASPX → WCF → BLL → DAL → SQL chain as a single coherent entity, with its known issues, its data contracts, and its change history.

**Workflow Context Files solve this:** A workflow-level context file captures the entire cross-layer flow as a single indexed artifact, with layer-by-layer summaries and cross-references.

---

### Gap 3: Deterministic + Semantic Hybrid Without Cloud

| Tool | Deterministic | Semantic | Local/Air-gapped |
|------|--------------|----------|-----------------|
| GraphRAG | No | Yes | Possible (complex) |
| Cursor | Yes (AST) | Yes | No |
| Aider | Yes | No | Yes |
| Continue.dev | Yes | Yes | Yes |
| Cody | Yes (SCIP) | Yes | Requires SG infra |
| CodeGraph | Yes | No | Yes |
| Copilot Workspace | Yes | Yes | No |
| **This project** | **Yes** | **Yes** | **Yes (Ollama)** |

Continue.dev is the closest competitor on this axis — but it lacks the workflow-unit concept and pre-computed context layer.

---

## Design Decisions Validated by This Analysis

| Decision | Validation |
|----------|------------|
| Pre-computed `.ai/` Context Files (mirror-path layout) | No existing tool does this. Eliminates per-query graph traversal. Mirror paths avoid filename collisions across multi-project solutions. |
| Workflow unit as first-class retrieval entity | No existing tool models cross-layer business workflows as a retrievable unit. **Shipped (May 2026):** `_workflows.json` — 1,970 entries, one per WebMethod, grouped by entity base into 1,804 groups, 922 full-chain (page→WebMethod→BLL→SVC→DAL→SQL). |
| 5-level hierarchical context expansion | Aider's compact signature map proves agents work better with summaries than full code. This project formalizes that into a queryable hierarchy. *(Levels 1–2 done in Phase 1; Levels 3–4 are Phase 2 LLM enrichment.)* |
| Deterministic naming-convention navigation as first retrieval layer | Aider validates the deterministic approach; this project extends it to enterprise naming patterns (.NET layer suffixes, etc.). |
| Local Ollama stack (Qwen3:8b + Qwen2.5-Coder:1.5b + nomic-embed-text) | Aider and Continue prove local-first is viable. Copilot and Cursor prove cloud-only is a dealbreaker for regulated industries. |
| JSON for storage; Qdrant deferred to Phase 4 | CodeGraph proves deterministic structural storage is sufficient for impact analysis. Qdrant added later for semantic search without architectural change. |
| Continue as IDE layer (experimentation/validation only) | Keeps IDE tooling lightweight during Phase 1–3; full agent interface via LangFlow pipeline. |

---

## What This Project Should Do That No Existing Tool Does

1. **Context File generation from source** *(Phase 1 — done)* — tree-sitter scanner reads each source file and generates a `.ai/{rel_path}.relationships.json` with all structural fields populated; semantic fields (`rh`, `br`, `dt`) left blank for Phase 2 LLM enrichment.

2. **Workflow discovery from naming conventions** *(Phase 1 — done, `_workflows.json` shipped)* — scanner detects `.NET`-style layer naming (`*BLL.cs`, `*DAL.cs`, `*Service.asmx`, `*.aspx`), builds the full execution graph, and emits one workflow entry per WebMethod grouped by entity base (CRUD verb-stripping + suffix normalization).

3. **Blast-radius via deterministic graph traversal** *(Phase 1 — `_execution_graph.json` + `_indexes.json` complete)* — instead of traversing raw source files, traverse the pre-computed execution graph. Impact analysis is a graph query, not LLM inference.

4. **Q&A capture loop** — when an agent discovers something true about a workflow that isn't in its Context File (e.g., via a debugging session), it writes a `qa` entry back to the Context File. Operational knowledge accumulates over time.

---

## References

- Microsoft GraphRAG: https://microsoft.github.io/graphrag/
- GraphRAG query modes: https://microsoft.github.io/graphrag/query/overview/
- AST-derived vs LLM-extracted graphs study: arXiv 2601.08773
- Aider repo-map: https://aider.chat/docs/repomap.html
- Aider repo-map blog: https://aider.chat/2023/10/22/repomap.html
- Continue.dev LanceDB architecture: https://lancedb.com/blog/the-future-of-ai-native-development-is-local-inside-continues-lancedb-powered-evolution/
- Continue.dev indexing internals: https://deepwiki.com/continuedev/continue/3.4-codebase-indexing
- Sourcegraph Cody context: https://sourcegraph.com/blog/how-cody-understands-your-codebase
- Cody research: arXiv 2408.05344
- Cursor secure indexing: https://cursor.com/blog/secure-codebase-indexing
- Cursor indexing architecture: https://towardsdatascience.com/how-cursor-actually-indexes-your-codebase/
- GitHub Copilot workspace context: https://code.visualstudio.com/docs/copilot/reference/workspace-context
- Copilot Workspace known issues: https://github.com/githubnext/copilot-workspace-user-manual/blob/main/known-issues.md
- CodeGraph: https://github.com/colbymchenry/codegraph
- CodePrism architecture: https://rustic-ai.github.io/codeprism/blog/graph-based-code-analysis-engine/
- GraphRAG cost analysis: https://medium.com/graph-praxis/the-graphrag-cost-cliff-how-33-000-became-33-in-eighteen-months-be1b0fbe37e4
