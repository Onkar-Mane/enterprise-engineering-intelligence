# System Architecture Diagrams

> Last updated: May 2026. Diagrams reflect the Phase 1 + Phase 5 (MCP) architecture.
> Phase 4 (Qdrant / vector search) is deferred — semantic paths are marked accordingly.

---

## 1. End-to-End Query Flow (current — Phase 1 + Phase 5 MCP)

```mermaid
flowchart TD
    A[Developer query] --> B[MCP Server]
    B --> C{Deterministic match?}
    C -->|Yes — naming / workflow| D[_workflows.json + _indexes.json]
    C -->|Yes — known symbol| E[_execution_graph.json]
    C -->|No match| F[search_semantic — Phase 4+ only]
    D --> G[Structured response]
    E --> G
    F --> G
    G --> H{Sufficient?}
    H -->|Yes| I[LLM reasoning over response]
    H -->|No| J[Raw code expansion — last resort]
    J --> I
```

---

## 2. Hybrid Retrieval Priority Order

```mermaid
flowchart TD
    subgraph "Phase 1-3 — Always available"
        D1[Naming / workflow match]
        D2[Graph traversal]
        D3[Per-file context lookup]
    end

    subgraph "Phase 4+ — Qdrant required"
        S1[Embeddings over rh + br fields]
        S2[Similarity search]
    end

    D1 --> M[Match found?]
    M -->|Yes| R[Return structured result]
    M -->|No| D2
    D2 --> M2[Match found?]
    M2 -->|Yes| R
    M2 -->|No| S1
    S1 --> S2 --> R
```

---

## 3. Phase 1 Scanner — Output Architecture

```mermaid
flowchart TD
    A[Source files scan] --> B[tree-sitter AST parse]
    B --> C[Per-file .relationships.json]
    B --> D[_execution_graph.json]
    B --> E[_indexes.json]
    B --> F[_shared_components.json]
    B --> G[_phase2_manifest.json]
    D --> H[Workflow builder]
    E --> H
    H --> I[_workflows.json]
    C --> J[.ai/ mirror-path layout]
    D --> J
    E --> J
    F --> J
    G --> J
    I --> J
```

---

## 4. Incremental Indexing Flow

```mermaid
flowchart TD
    A[Scan repository] --> B[Compare mtime + hash vs state.json]
    B --> C[Detect changed files]
    C --> D[Regenerate per-file .relationships.json]
    D --> E{Relationships changed?}
    E -->|Yes| F[Regenerate _execution_graph.json + _indexes.json + _workflows.json]
    E -->|No| G[Done]
    F --> G
```

---

## 5. Execution Chain — Layer Model

```mermaid
flowchart LR
    P[ASPX Page] --> WM[ASMX WebMethod]
    WM --> BLL[BLL Method]
    BLL --> SVC[WCF SVC Method]
    SVC --> DAL[DAL Method]
    DAL --> SQL[(SQL Table)]
```

Each hop is a typed edge in `_execution_graph.json`. The MCP tool `trace_execution_chain` walks this graph in either direction.
