# enterprise-engineering-intelligence
Workflow-aware engineering intelligence platform for large enterprise codebases using hybrid retrieval, hierarchical context systems, and progressive code expansion.

## Table of Contents

- [Vision](#vision)
- [Problem Statement](#problem-statement)
- [Proposed Direction](#proposed-direction)
- [Core Architecture Philosophy](#core-architecture-philosophy)
- [Retrieval Flow](#retrieval-flow)
- [Hybrid Retrieval Architecture](#hybrid-retrieval-architecture)
- [Engineering Digital Twin](#engineering-digital-twin)
- [Current Enterprise Target Architecture](#current-enterprise-target-architecture)
- [Context Intelligence System](#context-intelligence-system)
- [Progressive Context Expansion](#progressive-context-expansion)
- [Incremental Intelligence Refresh](#incremental-intelligence-refresh)
- [Current Prototype Stack](#current-prototype-stack)
- [Current Research Areas](#current-research-areas)
- [Repository Structure](#repository-structure)
- [Current Status](#current-status)
- [Long-Term Goals](#long-term-goals)
- [Important Note](#important-note)
- [Research Journal](#research-journal)
- [Architecture Evolution Log](#architecture-evolution-log)
- [License](#license)
- [Final Goal](#final-goal)


## Enterprise Engineering Intelligence Platform

## Workflow-Aware Repository Cognition for Large Enterprise Systems

---

## Vision

Enterprise software systems contain massive amounts of hidden architectural intelligence that traditional AI coding assistants fail to understand efficiently.

This project explores a new approach to engineering intelligence focused on:

* Workflow-aware retrieval
* Hierarchical context systems
* Deterministic + semantic hybrid retrieval
* Progressive code expansion
* Repository cognition
* Engineering digital twins
* Context-efficient reasoning

The long-term goal is to build a scalable engineering intelligence infrastructure capable of understanding and reasoning over extremely large enterprise software ecosystems.

---

## Problem Statement

Modern enterprise systems often contain:

* Millions of lines of code
* Distributed business logic
* Shared enterprise utilities
* Legacy architectures
* Complex workflow dependencies
* Large service chains
* Heavy database interaction
* Multi-layer application structures

Traditional AI coding systems typically operate using:

```text
Question
→ Embedding Search
→ Raw Code Chunks
→ LLM
```

This approach breaks down at enterprise scale because:

* Context windows overflow
* Retrieval quality degrades
* Architectural relationships are lost
* Business workflow understanding is weak
* Repeated reasoning becomes expensive
* Large methods become noisy
* Dependency tracing becomes unreliable

---

## Proposed Direction

Instead of treating repositories as random code chunks, this platform treats them as:

### Structured Engineering Systems

The architecture focuses on:

* Workflow intelligence
* Architectural conventions
* Deterministic navigation
* Repository relationships
* Context layering
* Incremental cognitive expansion

---

## Core Architecture Philosophy

### Never Load Entire Repositories

The system is designed around the principle that:

> Raw code should be the LAST retrieval layer.

The platform first retrieves:

* Workflow summaries
* Context intelligence
* Relationships
* Mappings
* Service chains
* Brief semantic summaries

Only if required does it expand into:

* Exact methods
* Exact regions
* Exact line ranges
* Raw source code

---

## Retrieval Flow

```text
Developer Question
        ↓
Workflow Detection
        ↓
Convention Mapping
        ↓
Context Registry Lookup
        ↓
Brief Context Retrieval
        ↓
Confidence Evaluation
        ↓
Detailed Context Expansion
        ↓
Raw Code Expansion
        ↓
LLM Reasoning
```

---

## Hybrid Retrieval Architecture

The platform combines:

### Deterministic Retrieval

Uses:

* Naming conventions
* Workflow mappings
* Service relationships
* Folder structures
* Architectural patterns

Example:

```text
LabelPrint.aspx
LabelPrint.js
LabelPrint.asmx
LabelPrintService.svc
LabelPrintDAL.cs
```

All treated as one workflow domain.

---

### Semantic Retrieval

Uses:

* Embeddings
* Summaries
* Keywords
* Similarity search
* Relationship expansion

---

### Why Hybrid Retrieval?

Enterprise systems already contain implicit architecture intelligence through:

* naming patterns
* service chains
* workflow conventions
* dependency structures

This project attempts to leverage that intelligence directly rather than relying only on embeddings.

---

## Engineering Digital Twin

This project explores the concept of an:

### Definition

A structured cognitive representation of the software ecosystem capable of understanding:

* workflows
* dependencies
* services
* repositories
* architectural relationships
* business logic boundaries
* operational behavior

The objective is to move beyond:

* generic code search
* naive repository RAG
* basic AI chat systems

and toward **reusable engineering intelligence infrastructure.**

---

## Current Enterprise Target Architecture

Current exploration targets large enterprise MES/SFS-style systems containing:

```text
ASP.NET WebForms
    ↓
ASMX Services
    ↓
WCF Services
    ↓
BLL
    ↓
DAL
    ↓
SQL Server
```

Hosted in:

* IIS
* Session-based enterprise environments

---

## Context Intelligence System

The platform introduces:

### Multi-Level Context Structures

Examples:

* Workflow Context
* Service Context
* DAL Context
* UI Context
* Common Entity Context
* Relationship Context
* Mapping Context

Each context layer contains:

* brief summaries
* detailed summaries
* relationships
* references
* workflow mappings
* line references
* dependency references

---

## Progressive Context Expansion

The system attempts to achieve:

* 80% reasoning from structured intelligence
* 20% reasoning from raw source code

This significantly reduces:

* token usage
* repeated cognition
* retrieval noise
* unnecessary code expansion

---

## Incremental Intelligence Refresh

Repository intelligence is designed to refresh incrementally.

The planned indexing flow:

```text
Scan Repository
    ↓
Compare Existing Metadata
    ↓
Detect Changed Files
    ↓
Refresh Impacted Context Only
    ↓
Refresh Relationships
    ↓
Refresh Embeddings
```

This prevents expensive full repository re-indexing.

---

## Current Prototype Stack

### Runtime
- Ollama 0.24.0

### Models
| Model | Role |
|-------|------|
| Qwen3:8b (Q4_K_M) | Primary semantic reasoning and indexing |
| Qwen2.5-Coder:1.5b | Autocomplete only |
| nomic-embed-text | Semantic embeddings (Phase 4 retrieval layer) |

### Orchestration
- LangFlow — visual multi-agent pipeline orchestration

### IDE Layer
- Continue (VS Code extension) — experimentation and validation only

### Index Storage
- Local filesystem (JSON) — structured context files in `.ai/`
- Qdrant (planned) — vector search on top of context files

---

## Current Research Areas

Active exploration areas:

* Workflow-aware retrieval
* Context compression
* Repository cognition
* Engineering digital twins
* Hybrid retrieval orchestration
* Deterministic architectural routing
* Enterprise repository indexing
* Context-efficient reasoning
* Progressive expansion systems

---

## Repository Structure

```text
architecture/
    proposals/
    diagrams/
    specifications/

research-journal/

whitepapers/

prototypes/

experiments/

docs/

```

---

## Current Status

### Completed

* Local inference setup (Ollama 0.24.0, Qwen3:8b, nomic-embed-text)
* Phase 1 — deterministic scanner (tree-sitter AST, production-ready)
  * Per-file `*.relationships.json` under `.ai/` mirroring project structure
  * `_execution_graph.json` — full page→webmethod→bll→svc→dal→table chain (12,413 nodes, 25,922 edges)
  * `_indexes.json` — exact and approximate lookups
  * `_shared_components.json` — reusable infra ranked by reference count
  * `_phase2_manifest.json` — per-project file-by-file work list for Phase 2
  * `_workflows.json` — **workflow-as-retrieval-unit** (1,970 entries, 1,804 groups, 922 full-chain)
  * Verified on real MES codebase: 237 pages, **1,970 ASMX WebMethods** (16 services), 16 WCF, 8 BLL, 16 DAL, 606 SQL tables, ~9,600 methods
  * BLL→SVC edge accuracy validated at 99.8% on real files
* Architecture diagrams (Mermaid) — system architecture, hybrid retrieval, incremental indexing
* Agent behavior specification (RFC format)
* Context file schema specification
* Context file generator specification
* Whitepaper extracted to Markdown
* Competitive landscape analysis

---

### Next — Phase 5 (MCP Server, design complete, build next)

A stateless MCP server wrapping `.ai/` JSON as callable tools. One server, all projects.
Tools exposed:

* `trace_execution_chain(project, page, method)` — full ASPX→DAL chain from `_execution_graph`
* `find_callers(project, dal_method)` — reverse lookup via `_indexes`
* `find_table_users(project, table_name)` — table→pages lookup
* `get_file_context(project, rel_path)` — reads `.relationships.json` for that file
* `list_workflow_files(project, workflow_group)` — returns chain from `_workflows.json`
* `search_semantic(project, query)` — Phase 4+ only, requires Qdrant

Query router: naming/workflow match first (deterministic, free). Vector search second. Raw code last.

---

### Phase 2 (LLM enrichment — not yet started)

File-by-file, region-by-region, driven by `_phase2_manifest.json`:

* `br` (brief) — 1-2 sentence summary per file/region
* `rh` (retrieval_hints) — keywords/phrases for vector search
* `dt` (detailed) — full business logic + `error_symptom_router` (3-5 symptom→cause entries, lives inside `dt` of each ASPX page)
* `lc` (local_common) — shared patterns/utilities
* `qa` — Q&A pairs for fine-tuning
* `cfg` — `gc_candidates` for shared-components catalog

---

## Long-Term Goals

Potential future directions:

* Multi-agent orchestration
* Repository graph systems
* Engineering memory layers
* DB schema intelligence
* CI/CD intelligence
* Impact analysis systems
* Production debugging intelligence
* Enterprise deployment architecture
* Distributed inference systems

---

## Important Note

This repository intentionally avoids publishing:

* proprietary enterprise code
* company business logic
* internal schemas
* confidential workflows

All examples and architectures are generalized for research and educational purposes.

---

## Research Journal

The `research-journal/` folder contains evolving thoughts, experiments, architectural explorations, and lessons learned while building scalable engineering intelligence systems.

The goal is to document not only successful ideas, but also:

* architectural tradeoffs
* retrieval failures
* indexing challenges
* scaling limitations
* enterprise AI constraints

---

---

## Architecture Evolution Log

This section documents real decisions, pivots, and lessons learned
during active development. A research project that only shows
successes is not honest research.

---

### May 2026 — Orchestration Stack Pivot

**What was tried:**
Initial stack used Continue.dev as the IDE layer with
qwen2.5-coder:7b as the primary model.

**What failed:**
Continue.dev cannot support multi-step autonomous agent workflows.
It is a chat interface, not an orchestration engine.
qwen2.5-coder:7b repeatedly output fake JSON tool calls as plain
text instead of executing tools correctly — causing failures:
`"The model provided reasoning but did not call required tools."`

**Root cause identified:**
The project is not building a normal coding assistant.
It is building a semantic intelligence platform requiring:
- reliable tool calling
- long autonomous workflows
- structured multi-step execution
- deterministic orchestration behavior

These requirements exceed what IDE-centric tools like Continue.dev
can provide.

**Decisions made:**
- Replaced Continue.dev with **Roo Code** for IDE layer
  (mode separation — Ask / Architect / Code — enforces correct
  agent behavior per task type)
- Replaced qwen2.5-coder:7b with **Qwen3:8b** for primary model
  (significantly better tool calling, semantic reasoning, and
  structured output reliability)
- Added **Gemma4:4b** as lightweight helper model for fast routing
  and low-cost retrieval decisions
- Added **Langroid** as the orchestration layer
  (hub-and-spoke multi-agent architecture, native Ollama support,
  no need to build orchestration from scratch)

**Key architectural realization:**
> The most valuable asset in this system is NOT the model.
> It is the hierarchical semantic indexing architecture.
> Models are replaceable. The index structure is the moat.

**Ollama version note:**
Qwen3 tool calling had a known parsing bug fixed in Ollama v0.17.6.
Current version is 0.24.0 — bug is resolved. No workarounds needed.

---

### May 2026 — Phase 1 Complete + Second Stack Pivot

**What was delivered:**
Phase 1 deterministic scanner is production-ready. Tree-sitter AST extraction
across all layers (ASPX, JS, ASMX, WCF, BLL, DAL, SQL). Full execution graph
traced. Verified on real MES codebase: 237 pages, 1,970 WebMethods (16 services), ~9,600 methods,
606 SQL tables. Runtime: ~1m25s for the full project (was 9.5m before JS deep-parse fix).

**Stack corrections:**
- Replaced Langroid with **LangFlow** for orchestration
  (visual pipeline design fits multi-agent workflow better; no custom Python runtime)
- Replaced Roo Code with **Continue** for IDE layer
  (Continue used for experimentation and validation only — agent interface goes
  through LangFlow pipelines, not IDE modes)
- Removed Gemma4:4b helper model
  (Qwen3:8b handles routing directly; helper model added latency without benefit)

**Architecture correction — output path:**
- Replaced flat `.ai-memory/context/[filename].json` with `.ai/` mirror-path layout
  (`{ProjectName}/{mirrored/path}/{file}.relationships.json`)
- Flat layout caused filename collisions across multi-project solutions
- Mirror paths are collision-free and directly navigable

**Key numbers locked in:**
- Deterministic scan: ~3,000 files/min (CPU-only, no LLM)
- LLM enrichment (Phase 2): ~2,000 files/hr on RTX 4050

---

## License

MIT License

---

## Final Goal

Build a scalable engineering intelligence infrastructure capable of understanding extremely large enterprise systems through:

* workflow cognition
* deterministic retrieval
* progressive context expansion
* architectural intelligence
* hybrid retrieval systems
* context-efficient reasoning

rather than relying solely on:

* large prompts
* raw code chunking
* naive vector search
* brute-force context loading
