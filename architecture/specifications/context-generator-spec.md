# Context File Generator Specification

---

## 1. Purpose and Goals

The **Context File Generator** is a command-line tool that scans enterprise software repositories to automatically generate structured JSON context files describing key components and their workflow relationships. These files are consumed by an AI-driven engineering intelligence platform to enable context-aware reasoning, code navigation, and workflow suggestions.

### Key Goals:
- **Automatically infer** high-level workflow boundaries from file naming and directory structure.
- **Classify** components into categories (UI, Service, BLL, DAL) based on file type, naming conventions, and location.
- **Generate structured, version-control-friendly JSON context** files for use by downstream AI agents.
- **Support incremental updates** to minimize performance overhead in large repositories.
- **Be configurable and extensible** to support diverse enterprise codebase layouts.

---

## 2. Input

### Scanned File Types
The generator scans the following source file types:
- `.aspx` — ASP.NET Web Forms (UI layer)
- `.cs` — C# source files (BLL, Service, DAL, etc.)
- `.asmx` — ASMX web services
- `.svc` — WCF service endpoints
- `.js` — JavaScript files (client-side UI logic)
- `.sql` — SQL scripts (stored procedures, table definitions)

### Folder Structure Assumptions
The tool assumes a conventional enterprise .NET project layout. Examples:
```
/ProjectRoot
  /UI/
    /Pages/             → .aspx, .js
  /Services/
    /ASMX/              → .asmx
    /WCF/               → .svc
  /BusinessLogic/
    /Managers/          → .cs (BLL)
  /DataAccess/
    /Repositories/      → .cs, .sql
  /Common/
    /Utilities/         → Shared .cs
```

While flexible, the tool uses folder paths as a signal in classification when naming conventions are ambiguous.

---

## 3. Output

The generator produces **one per-file record** plus **four aggregate files**, all written under `.ai/` mirroring the project folder structure.

> **Previous design (stale):** Four workflow-grouped files (`workflow-context.json`, `service-context.json`, `dal-context.json`, `ui-context.json`). This caused flat-file collisions in multi-project solutions and was replaced by the mirror-path layout below.

### 3.1 Per-file: `{rel_path}.relationships.json`

One file per indexed source file, located at `.ai/{ProjectName}/{mirrored/path}/{filename}.relationships.json`.

Contains two layers (see context-file-schema-v1.md):
- **Layer 1 (Phase 1 — deterministic):** `rel_path`, `type`, `layer`, `hash`, `total_lines`, `contents.regions`, method index, call relationships, `batch_plan`, `conflicts`
- **Layer 2 (Phase 2 — LLM-enriched):** `rh`, `br`, `dt`, `lc`, `qa`, `cfg` — added file-by-file during Phase 2; absent until enrichment runs

**Minimal Phase 1 example:**
```json
{
  "file": "LabelPrintDAL.cs",
  "rel_path": "iPAS_Service/DataAccess/LabelPrintDAL.cs",
  "type": "dal",
  "layer": "backend",
  "hash": "b7e2a1...",
  "total_lines": 1840,
  "contents": {
    "regions": [
      {
        "name": "Label Queries",
        "ln": "1-920",
        "mc": 14,
        "mi": [
          { "name": "GetLabelFields", "ln": "45-89", "sql_tables": ["caufv_hdr", "caufv_pos"] }
        ]
      }
    ]
  },
  "dal_sql_tables": ["caufv_hdr", "caufv_pos", "zfi_label_config"],
  "batch_plan": [
    { "batch": 1, "regions": ["Label Queries"], "methods": ["GetLabelFields", "GetBulkLabels"], "est_lines": 420 }
  ],
  "conflicts": []
}
```

### 3.2 `_execution_graph.json`

Fully traversable graph covering the complete execution chain:
`page → webmethod → bll → svc → dal_class → sql_table`

Uses a single node scheme and single edge format `{source, target, relation}`.
Enables deterministic impact analysis — "what breaks if I change this DAL method?" — via graph traversal, no LLM required.

### 3.3 `_indexes.json`

Exact and approximate lookup tables:
- `page_to_webmethods` — which WebMethods does a page call?
- `webmethod_to_pages` — which pages call this WebMethod?
- `page_to_services` — ASMX/WCF services used by a page
- `page_to_dal_classes` — DAL classes reachable from a page
- `dalclass_to_tables` — SQL tables a DAL class touches
- `page_to_tables` — approximate (class-level; flagged in file — use `page_to_dal_classes + dalclass_to_tables` for precision)

### 3.4 `_shared_components.json`

Reusable infrastructure components ranked by reference count across all pages.
Example: `CommonBLL.ValidateSiteID` referenced by 212 pages — a prime `gc_candidate`.

### 3.5 `_phase2_manifest.json`

Per-project ordered work list for Phase 2 LLM enrichment. Each entry is a file record containing `rel_path`, `contents`, and `batch_plan` — everything the LLM needs in one place. JS entries carry a `referenced` flag; orphan/vendor JS is excluded from the manifest (deep-parse skipped at scan time to keep runtime fast).

### 3.6 `_workflows.json`

The platform's #1 differentiator — workflow-as-retrieval-unit. One entry per ASMX WebMethod (= one atomic business operation), grouped by entity base into clusters that represent the full CRUD feature.

Stats from the reference codebase: 1,970 workflow entries, 1,804 groups, 922 full-chain (depth 4+), 82 no-downstream utility endpoints, 850 BLL-only (CommonBLL infrastructure or external compiled DLLs — surfaced honestly, not flagged as failures).

Full schema in `workflows-schema.md`.

All output is written under `.ai/` and is excluded from version control via `.gitignore`.

---

## 4. Detection Logic

### 4.1 Workflow Grouping Detection
The system infers workflows by **normalizing and matching base names** across file types.

- **Step 1**: Extract base name (e.g., `LabelPrint` from `LabelPrint.aspx`, `LabelPrintService.svc`).
- **Step 2**: Normalize using:
  - Strip common suffixes: `Service`, `Manager`, `Repository`, `Page`, `Form`.
  - Normalize casing to PascalCase.
- **Step 3**: Group files with matching base names across `.aspx`, `.svc`, `.asmx`, `.cs`, and `.js`.

**Match Example**:
- `LabelPrint.aspx` → `LabelPrint`
- `LabelPrintService.svc` → `LabelPrint`
- `LabelPrintController.cs` → `LabelPrint`
→ All grouped into workflow "LabelPrint".

**Fallback**: If naming is not aligned, use directory proximity (e.g., files in `/Features/LabelPrint/`) as group signal.

### 4.2 Component Type Detection

| File Type | Detection Rule |
|---------|----------------|
| `.aspx` | Always classified as `UI` |
| `.js`   | Classified as `UI` if in `/Scripts/`, `/UI/`, or referenced by `.aspx` |
| `.asmx` | Classified as `Service` of type `ASMX` |
| `.svc`  | Classified as `Service` of type `WCF` |
| `.cs`   | Use file name and path: <br> - Contains `Service`, `Manager` → `BLL` <br> - Contains `Repository`, `Dao` → `DAL` <br> - Else: `BLL` (default) |
| `.sql`  | Classified as `DAL` source; linked via naming or annotations |

---

## 5. Incremental Refresh

To support efficient use in CI/CD or IDE integrations, the generator supports **incremental regeneration**:

### Mechanism:
- On first run, scan entire repository and generate full context files.
- Store a `.contextgen/state.json` file with:
  - File paths and their last-modified timestamps (from `os.stat()`).
  - Hash of file content (SHA-256 of first 8KB for large files).
  - Output file generation timestamps.

### On Subsequent Runs:
1. Compute difference between current file mtime/hash and stored state.
2. Identify **changed files** (modified, added, deleted).
3. Determine **affected files** by:
   - Any changed file → regenerate its `{rel_path}.relationships.json`.
   - Any changed file whose relationships touch the execution graph → regenerate `_execution_graph.json` and `_indexes.json`.
4. Only regenerate affected JSON files.
5. Update `state.json` with new timestamps and hashes.

### Options:
- `--clean`: Delete all `.ai/` output and run a full rescan. Required when source files are deleted (incremental runs overwrite but never remove stale records for deleted files).
- Default (no flag): Incremental — overwrite records for changed files only.

> **Known limitation:** Incremental runs do not delete records for source files that were removed. Run with `--clean` after any bulk file deletion or project restructure.

---

## 6. Configuration

The tool uses a `contextgen.yaml` config file (optional; defaults applied if missing).

### Schema:
```yaml
# Root directory to scan (default: current dir)
scan_root: "./src"

# Directories to exclude (supports glob patterns)
exclude_paths:
  - "**/obj/**"
  - "**/bin/**"
  - "**/node_modules/**"
  - "Legacy/**"

# Output directory for generated JSON files
output_dir: "./.contextgen/output"

# Naming patterns for workflow base name extraction
naming_patterns:
  service_suffixes: ["Service", "Manager", "Handler"]
  dal_suffixes: ["Repository", "Dao", "DataProvider"]
  ui_suffixes: ["Page", "Form", "View"]

# File inclusion overrides (if default types not sufficient)
file_types:
  include:
    - "*.aspx"
    - "*.svc"
    - "*.asmx"
    - "*.cs"
    - "*.js"
    - "*.sql"
  exclude:
    - "Generated/*.cs"

# Enable debug logging
debug: false
```

---

## 7. CLI Interface

### Installation:
```bash
pip install contextgen
```

### Example Commands:

**Basic scan with defaults:**
```bash
contextgen generate
```

**With custom config:**
```bash
contextgen generate --config ./configs/prod.yaml
```

**Incremental scan (default):**
```bash
contextgen generate --incremental
```

**Force full regeneration:**
```bash
contextgen generate --full-scan
```

**Specify output directory:**
```bash
contextgen generate --output-dir ./artifacts/context
```

**List detected workflows (dry-run):**
```bash
contextgen list --type workflows
```

**Print tool version:**
```bash
contextgen --version
```

---
## 8. Multi-Project Solution Handling

### 8.1 Problem Statement

Enterprise ASP.NET solutions present unique challenges for context generation due to their complex structure and interconnected nature. Common issues include:

- **Large Solutions**: Enterprise `.sln` files often contain 10–50 or more projects, requiring deep parsing across diverse locations.
- **Shared Projects**: Projects such as shared libraries or framework layers (e.g., BLL or DAL) are frequently referenced across multiple solutions, introducing redundancy if not carefully managed.
- **Namespace Overlap**: Projects in different solutions can have overlapping namespaces, potentially leading to ambiguous references or inaccurate indexing.
- **Single-Project Assumptions**: Traditional context generators designed for smaller or single-project solutions break when required to support enterprise-scale `.sln` files with cross-project and shared dependencies.

These challenges necessitate enhancements to the generator's ability to support multi-project workflows and accurately maintain context fidelity across interconnected solutions.

---

### 8.2 Solution Discovery

The generator implements an enhanced `.sln` scanning and parsing mechanism to address multi-project solution complexity:

- **Solution Scanning**:
  - The generator recursively scans the target directory for `.sln` files. It filters and prioritizes these based on predefined inclusion/exclusion rules.
- **Project Reference Extraction**:
  - The `.sln` parser extracts all project references, including interdependencies defined within solution files.
  - References to `.csproj`, `.vbproj`, and project-specific metadata such as target frameworks or output paths are cataloged.
- **Nested Solutions**:
  - If solution folders or nested solutions exist, the generator iterates through their hierarchy and includes these in the reference map.
  - Flexible configuration options allow users to either expand nested solutions or treat them independently.

---

### 8.3 Shared Project Detection

To mitigate duplication and redundancy, the generator applies logic to detect shared projects and manage them appropriately:

- **Shared Project Identification**:
  - Projects referenced by multiple `.sln` files are flagged as shared. The generator does this by comparing project GUIDs, paths, and names across the scanned `.sln` files.
- **Avoiding Duplicate Indexing**:
  - Shared projects are indexed a single time and annotated in the context files with a shared flag. This prevents repeated indexing for the same project across different `.sln` files.
- **Special Handling of Core Projects**:
  - Shared projects in layers such as BLL or DAL are treated as global components. Metadata such as usage frequency and dependency counts across solutions is appended to their context files for enhanced traceability.
  - Example annotation for shared projects:

```json
    {
      "project_id": "BLL_Core",
      "shared_flag": true,
      "shared_solutions": ["SolutionA.sln", "SolutionB.sln"],
      "indexed_only_once": true
    }
```

---

### 8.4 Cross-Solution Workflow Linking

Some enterprise workflows span projects located in different solutions. The generator supports cross-solution workflows through forward and reverse dependency mapping:

- **Cross-Solution Dependencies**:
  - When dependencies link projects across solutions, the generator records these in context files as explicit cross-solution references, allowing complete workflow traceability.
- **Workflow ID Naming**:
  - Cross-solution workflows are assigned unique IDs using the following convention:
    - `SolutionName1_ProjectNameX → SolutionName2_ProjectNameY_WorkflowID`
  - Example:

```json
    {
      "workflow_id": "SolutionA_ProjectA → SolutionB_ProjectB_Workflow123",
      "cross_solution_flag": true,
      "details": {
        "source_project": "ProjectA",
        "destination_project": "ProjectB",
        "dependency_type": "MethodInvocation"
      }
    }
```

- **Context File Updates**:
  - Inter-solution dependencies are appended to the relevant workflow or method context files under a `CrossSolutionDependencies` section.

---

### 8.5 Configuration

To support flexible integrations across enterprise environments, the generator provides configuration options for selecting and prioritizing `.sln` files.

- **Solution Inclusion/Exclusion**:
  - Define specific `.sln` files to include or exclude during the scanning phase.
  - Example:

```yaml
    solution_inclusion:
      - "MainSolution.sln"
      - "SupportSolution.sln"

    solution_exclusion:
      - "LegacySolution.sln"
```

- **Solution Priority**:
  - When conflicts exist (e.g., shared project ambiguity), solutions can be prioritized using a ranking system.
  - Example:

```yaml
    priority_order:
      - solution: "MainSolution.sln"
        rank: 1
      - solution: "SupportSolution.sln"
        rank: 2
```

These configurations allow developers to avoid indexing unnecessary or legacy solutions and ensure the generator prioritizes critical workflows correctly.
