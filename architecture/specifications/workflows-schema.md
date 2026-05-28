# `_workflows.json` Schema

> **Status:** Shipped (May 2026). The #1 architectural differentiator of this platform — workflow-as-retrieval-unit. One entry per ASMX WebMethod = one atomic business operation traceable across all layers.

---

## 1. Why workflows are the retrieval unit

Standard code-intelligence tools return individual files or methods. That works for greenfield projects where a feature lives in one file. It breaks down for enterprise WebForms codebases where one business operation spans seven layers: ASPX → JS → ASMX WebMethod → BLL → WCF SVC → DAL → SQL table.

Asking "show me how MoldingSchedule works" should return the *workflow*, not a single file. `_workflows.json` makes the workflow itself a first-class retrieval entity.

### Philosophy

Traditional repository search treats files independently. This system treats workflow chains as unified semantic structures — each WebMethod is one atomic business capability, one deterministic retrieval domain, one unit of reasoning. The platform searches by intent, not by filename.

### Benefits of workflow-as-retrieval-unit

- **Impact analysis** — one query surfaces every layer a change will touch
- **Dependency tracing** — follow the chain from page to SQL table without reading code
- **Cross-layer navigation** — jump from ASPX to DAL in one tool call
- **Workflow reasoning** — the AI agent reasons about a feature, not a method
- **Retrieval optimization** — the common case hits `_workflows.json` without any vector search

### Future directions

Workflow mappings are the foundation for:

- Engineering graph systems (architectural drift detection)
- Debugging intelligence (trace runtime failures to workflow boundaries)
- Impact simulations (predict blast radius before committing)
- Cross-solution dependency analysis (Phase 8+ concept)

---

## 2. File location

```
.ai/_workflows.json          ← one file at the .ai/ root, covers all projects
```

The scanner emits this once per full run. It is regenerated whenever any project's relationships change.

---

## 3. Top-level shape

```json
{
  "stats": { ... },
  "workflows": [ ... ],   // one entry per WebMethod
  "groups": [ ... ]       // workflows clustered by entity base + service
}
```

---

## 4. `stats` — health snapshot

```json
{
  "total_workflows": 1970,
  "total_groups": 1804,
  "full_chain": 922,          // depth 4+ (page → wm → bll → svc → dal)
  "no_downstream": 82,         // utility/enum endpoints — expected
  "bll_only": 850,             // stops at BLL (CommonBLL infra or external DLL)
  "bll_only_breakdown": {
    "common_bll": 935,         // CommonBLL.ValidateUserPrivilege etc. — always expected
    "external_bll": [          // .cs source not in this repo (compiled-in DLLs)
      "StagingOrderBLL", "QualityBLL", "KanbanBLL",
      "VPromaxBLL", "DashboardBLL", "ExternalWarehouseBLL",
      "PreProcessBLL", "ReportBLL"
    ]
  }
}
```

> **Why bll-only is surfaced honestly:** Many workflows legitimately stop at the BLL layer because they call into infrastructure (CommonBLL) or external compiled DLLs whose source is not in the repo. Reporting these as failures would be dishonest; they are valid endpoints.

---

## 5. `workflows[]` — one entry per WebMethod

```json
{
  "id": "iPAS_MoldService/ScheduleUnScheduleOrderInfo",
  "name": "ScheduleUnScheduleOrderInfo",
  "service": "iPAS_MoldService",
  "entity_base": "ScheduleUnScheduleOrder",
  "pages": ["Mold/ManageMoldingSchedule.aspx"],
  "page_count": 1,
  "chain": {
    "webmethod": "webmethod:iPAS_MoldService/ScheduleUnScheduleOrderInfo",
    "bll_methods": ["bll:MoldingBLL.ScheduleUnScheduleOrderInfo"],
    "svc_methods": ["svc:MoldingService.ScheduleUnScheduleOrderInfo"],
    "dal_methods": ["dal:MoldingDAL.UpdateScheduleInfo"],
    "dal_classes": ["MoldingDAL", "CommonDAL"],
    "sql_tables": ["LDB1_CAUFV_HDR", "LDB1_MOLDORDER_RESOURCEID_SEQUENCE"],
    "stored_procedures": []
  },
  "chain_depth": 5
}
```

### Field reference

| Field | Meaning |
|---|---|
| `id` | Stable unique id — `{service}/{webmethod_name}` |
| `name` | The WebMethod name as declared in `.asmx.cs` |
| `service` | ASMX service file base name |
| `entity_base` | Derived business entity (see §7 — verb-stripping algorithm) |
| `pages` | ASPX pages that call this WebMethod (from `_indexes.webmethod_to_pages`) |
| `page_count` | `len(pages)` — used for ranking; high count = important workflow |
| `chain.webmethod` | Graph node id for the WebMethod |
| `chain.bll_methods` | BLL method nodes called by the WebMethod |
| `chain.svc_methods` | WCF SVC method nodes called by the BLL |
| `chain.dal_methods` | DAL method nodes called by the SVC |
| `chain.dal_classes` | DAL classes touched (class-level, exact) |
| `chain.sql_tables` | SQL tables touched (class-level — see precision note below) |
| `chain.stored_procedures` | Stored procs invoked (when statically detectable) |
| `chain_depth` | Number of layers the chain traverses |

### Precision note on `sql_tables`

`sql_tables` is **class-level**, not per-method. The DAL class as a whole touches these tables — the specific DAL method in this workflow may touch a subset. For precise per-method lookup, use `_indexes.json` (`dalclass_to_tables` + per-method `dal_sql_tables`). This limitation is inherited from the SQL extraction approach and is documented in `edge-cases.md` limitation #2.

---

## 6. `groups[]` — entity-level clustering

Groups cluster all CRUD-style workflows for one business entity, so a single retrieval returns the full feature.

```json
{
  "group_id": "iPAS_SOPWebService/MoldingSOPTemplate",
  "entity_base": "MoldingSOPTemplate",
  "service": "iPAS_SOPWebService",
  "workflow_ids": [
    "SaveMoldingSOPTemplateInfo",
    "ApproveMoldingSOPTemplate",
    "GetMoldingSOPTemplateList",
    "DeleteMoldingSOPTemplate"
  ],
  "workflow_count": 4,
  "all_pages": ["Mold/AddSOPTemplateInfo.aspx", "Mold/ManageSOPTemplate.aspx"],
  "all_tables": ["LDB1_SOPHEADER", "LDB1_SOPPHASEINSTRUCTION"]
}
```

| Field | Meaning |
|---|---|
| `group_id` | `{service}/{entity_base}` |
| `entity_base` | The shared business entity that ties these workflows together |
| `workflow_ids` | Member workflow names (look up full entries in `workflows[]`) |
| `workflow_count` | `len(workflow_ids)` |
| `all_pages` | Union of pages across all member workflows |
| `all_tables` | Union of SQL tables across all member workflows |

Groups are sorted by total page count descending — the most-used features come first.

---

## 7. `entity_base` derivation algorithm

The `entity_base` strips CRUD verb prefixes and noise suffixes from the WebMethod name to find the underlying business entity:

1. **Strip leading CRUD verb prefixes** (case-insensitive): `Get`, `Save`, `Update`, `Insert`, `Delete`, `Add`, `Remove`, `Download`, `Upload`, `Check`, `Mark`, `Approve`, `Reject`, `Cancel`, `Submit`, `Process`, `Validate`, `List`, `Search`, `Fetch`, `Load`.
2. **Strip trailing noise suffixes**: `List`, `Info`, `Detail`, `Details`, `Data`, `Result`, `Results`, `Records`.
3. **Fallback:** If fewer than 3 characters remain after stripping, return the original method name unchanged.

### Worked examples

| WebMethod name | `entity_base` |
|---|---|
| `GetMoldingSOPTemplateList` | `MoldingSOPTemplate` |
| `SaveMoldingSOPTemplateInfo` | `MoldingSOPTemplate` |
| `ApproveMoldingSOPTemplate` | `MoldingSOPTemplate` |
| `DeleteMoldingSOPTemplate` | `MoldingSOPTemplate` |
| `Ping` | `Ping` (fallback — too short after stripping) |
| `GetData` | `GetData` (fallback — `Data` is suffix-stripped, leaving `Get` which is too short) |

---

## 8. Generation order

1. Phase 1 scanner emits per-file `.relationships.json` and `_execution_graph.json`.
2. Workflow builder reads `_indexes.json` (`webmethod_to_pages`) and walks the graph from each WebMethod node downward.
3. Entity base is computed per workflow.
4. Groups are formed by `(service, entity_base)`.
5. Stats are computed and written to the top-level `stats` block.

No LLM is involved. The full `_workflows.json` is reproducible from the same input codebase.

---

## 9. Known limitations

1. **SQL tables are class-level (over-inclusive).** See §5 precision note. Use `_indexes.json` for per-method precision.
2. **BLL-only workflows are not failures.** They legitimately stop at the BLL layer when calling CommonBLL or external compiled DLLs. Surfaced in `stats.bll_only_breakdown`.
3. **SVC→SVC calls not yet traced.** Currently only BLL→SVC `calls_service` edges exist in the graph. Workflows that route through a chain of SVCs will currently show only the first SVC hop.
4. **Stored procedures** are detected only when the SP name is a static string literal. Dynamically constructed SP names (string concatenation, config-driven) are not captured.

---

## 10. Consumers

`_workflows.json` is read by:

- `mcp-server-spec.md` → `list_workflow_files` tool
- `agent-behavior-spec.md` → `ImpactAnalysisAgent` (for "show the full feature" queries)
- Future Phase 2 LLM enrichment — workflows are the unit of batching, so one LLM call summarizes one workflow (not one file)
