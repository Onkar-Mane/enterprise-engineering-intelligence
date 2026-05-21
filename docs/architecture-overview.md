# Architecture Overview

## High-Level Objective

Build a workflow-aware engineering intelligence platform capable of understanding and reasoning over extremely large enterprise software systems.

The platform focuses on:

- repository cognition
- workflow intelligence
- deterministic retrieval
- semantic retrieval
- context-efficient reasoning
- progressive code expansion

---

# High-Level Retrieval Flow

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

# Architectural Layers

## 1. Raw Repository Layer

Contains:
- source code
- SQL
- configs
- frontend assets
- resources

This layer is NOT directly loaded into prompts.

---

## 2. Repository Intelligence Layer

Contains:
- workflow mappings
- service relationships
- deterministic routing
- entity references
- repository relationships

---

## 3. Context Intelligence Layer

Contains:
- brief summaries
- detailed summaries
- references
- line mappings
- dependency mappings

---

## 4. Retrieval Orchestration Layer

Responsible for:
- workflow detection
- context routing
- confidence evaluation
- progressive expansion

---

## 5. Dynamic Expansion Layer

Loads:
- exact methods
- exact regions
- exact line ranges
- related workflow chains

only when required.

---

# Core Principle

Raw code should be treated as the LAST retrieval layer.