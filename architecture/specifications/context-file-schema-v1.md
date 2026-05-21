# Context File Schema — Version 1.0

Specification for the structured JSON context files produced
by the hierarchical semantic indexing pipeline.

Status: Finalized for Phase 1 implementation
Last updated: May 2026

---

## Purpose

Each indexed source file produces one context file.
The context file is the primary retrieval artifact — the agent
reads this instead of the raw source code.

Raw code is opened only when the context file is insufficient.
This is the foundational principle of the entire system.

---

## File Location

```
.ai-memory/
  context/
    [filename].json          ← one per indexed file
  raw-docs/
    [filename]/
      [region-name].md       ← Stage 1 output, one per region
  global-common/
    GlobalCommon.json        ← entities shared across files
```

`.ai-memory/` is excluded from version control via `.gitignore`.

---

## Retrieval Order

The agent MUST follow this order. Never skip levels.

```
1. rh   — retrieval hints       (is this file relevant?)
2. contents                     (which region? which method?)
3. br   — brief                 (what does this region do?)
4. dt   — detailed              (how does it work exactly?)
5. raw code at exact line       (last resort only)
```

---

## Key Abbreviations (legend)

All context files begin with this legend block.
Abbreviations apply to JSON keys only — never to values,
method names, variable names, or file paths.

| Key | Full Name |
|-----|-----------|
| fp | physical_path |
| lc | local_common |
| gc_ref | global_common_ref |
| br | brief |
| dt | detailed |
| rh | retrieval_hints |
| ln | lines |
| mc | method_count |
| ls | light_summary |
| ds | deep_context_ref |
| mi | method_index |
| mn | magic_numbers |
| sv | shared_variables |
| sp | shared_patterns |
| qa | questions_answered |
| cfg | conflicts |

---

## Full Schema

```json
{
  "_legend": {
    "fp": "physical_path",
    "lc": "local_common",
    "gc_ref": "global_common_ref",
    "br": "brief",
    "dt": "detailed",
    "rh": "retrieval_hints",
    "ln": "lines",
    "mc": "method_count",
    "ls": "light_summary",
    "ds": "deep_context_ref",
    "mi": "method_index",
    "mn": "magic_numbers",
    "sv": "shared_variables",
    "sp": "shared_patterns",
    "qa": "questions_answered",
    "cfg": "conflicts"
  },

  "file": "LabelPrintService.cs",
  "fp": "iPAS_Service/Services/LabelPrintService.cs",
  "type": "service",
  "layer": "backend",
  "last_indexed": "2026-05-01",
  "hash": "a3f9c2...",
  "total_lines": 40240,

  "rh": [
    "How does label generation work?",
    "What happens when the print queue is full?",
    "How is batch print different from single print?",
    "What validations run before a label prints?"
  ],

  "contents": {
    "gc_ref": "GlobalCommon.json#LabelPrint",
    "regions": [
      {
        "name": "Initialization",
        "ln": "1-280",
        "mc": 8,
        "ls": "Bootstraps service dependencies and loads print config on startup.",
        "ds": "dt.Initialization",
        "mi": [
          {
            "name": "Initialize",
            "ln": "45-89",
            "ls": "Loads PrintConfig and validates printer connection"
          }
        ]
      },
      {
        "name": "Label Generation",
        "ln": "281-1100",
        "mc": 24,
        "ls": "Validates order state, builds label payload, supports batch and single modes.",
        "ds": "dt.Label Generation",
        "mi": [
          {
            "name": "GetLabelData",
            "ln": "345-389",
            "ls": "Fetches label fields from caufv_hdr for one order"
          },
          {
            "name": "ValidateOrder",
            "ln": "390-431",
            "ls": "Checks order status before allowing print"
          },
          {
            "name": "GetLabelDataBulk",
            "ln": "432-510",
            "ls": "Batch fetch path — used when batchMode is true"
          }
        ]
      }
    ]
  },

  "lc": {
    "mn": [
      {
        "val": 4,
        "meaning": "order confirmed — only state where print is allowed",
        "src": "comment line 445"
      },
      {
        "val": 500,
        "meaning": "batch size limit — above this timeout occurs silently",
        "src": "developer answer"
      },
      {
        "val": 30,
        "meaning": "printer timeout in seconds — not configurable per label type",
        "src": "method top summary line 892"
      }
    ],
    "sv": [
      {
        "name": "_printConfig",
        "type": "PrintConfigDTO",
        "used_in": ["Initialization", "Label Generation", "Print Queue"]
      }
    ],
    "sp": [
      {
        "pattern": "null check before DTO return",
        "seen_in": ["GetLabelData", "GetQueueStatus", "GetPrinterInfo"]
      }
    ],
    "gc_candidates": [
      {
        "name": "PrintConfigDTO",
        "reason": "likely used in other print-related services",
        "status": "pending_developer_decision"
      }
    ]
  },

  "br": {
    "Initialization": {
      "ln": "1-280",
      "summary": "Bootstraps all service dependencies on startup. Loads printer configuration and validates that the printer connection is available before accepting requests."
    },
    "Label Generation": {
      "ln": "281-1100",
      "summary": "Validates order state then builds label payload from DB. Supports single and batch modes. Batch path uses a different SQL fetch and silently fails above 500 units."
    }
  },

  "dt": {
    "Label Generation": {
      "ln": "281-1100",
      "workflow": "GetLabelData → ValidateOrder → BuildLabelPayload → SubmitToPrinter",
      "key_conditions": [
        "status == 4 means order is confirmed — only state that passes ValidateOrder (line 445)",
        "batchMode == true triggers GetLabelDataBulk() instead of GetLabelData() (line 512)"
      ],
      "edge_cases": [
        "Batch silently fails above 500 units — no exception thrown, returns partial result (line 678)",
        "null orderId skips validation entirely and returns empty DTO (line 392)"
      ],
      "methods": [
        {
          "name": "GetLabelData",
          "ln": "345-389",
          "purpose": "Fetches all label fields for one order from caufv_hdr",
          "params": {
            "orderId": "production order ID",
            "batchMode": "true = use bulk fetch path"
          },
          "returns": "LabelDataDTO or null if order not found",
          "calls": [
            "LabelPrintDAL.GetLabelFields ln:367",
            "ValidateOrder ln:355"
          ],
          "mn": [
            { "val": 4, "meaning": "confirmed status", "ln": 371 }
          ]
        }
      ]
    }
  },

  "qa": [
    {
      "q": "What does status == 4 mean in ValidateOrder?",
      "a": "Order is confirmed — the only state in which printing is allowed",
      "ln": 445,
      "by": "developer",
      "date": "2026-05-01"
    }
  ],

  "cfg": [
    {
      "desc": "GetLabelData exists with two signatures — with and without batchMode parameter",
      "locs": ["ln:345", "ln:567"],
      "resolution": "ln:567 is current — ln:345 is legacy, kept for backward compatibility with old callers",
      "by": "developer"
    }
  ]
}
```

---

## Design Rules

1. **Never embed raw code** inside context files — only summaries,
   line references, and pointers
2. **Abbreviate keys only** — never values, method names, or paths
3. **Every method entry must have exact line numbers**
4. **gc_candidates are flagged, never auto-promoted** — developer
   decides what goes into GlobalCommon
5. **qa entries are permanent** — once a business logic question
   is answered it is never asked again
6. **Agent must show draft before saving** — developer approves
   every context file before it is written to disk

---

## Two Common Layers

**LC — Local Common**
Entities shared across regions within one file.
Stored inside that file's context JSON under `lc`.

**GC — Global Common**
Entities shared across multiple service files.
Stored separately in `GlobalCommon.json`.
Agent flags candidates under `lc.gc_candidates`.
Developer decides during the workflow linking pass.

---

## Magic Number Sources

The agent checks for magic number documentation in this order:

1. Inline comment on the condition line — `== 4 // order confirmed`
2. Developer's method-top summary block
3. Ask the developer directly (answer stored in `qa`)

If source is `"comment line N"` and that comment is later removed,
the update indexing pass flags it for re-verification.

---

## Schema Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | May 2026 | Initial finalized schema |