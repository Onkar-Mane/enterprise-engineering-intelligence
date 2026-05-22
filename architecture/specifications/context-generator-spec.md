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

The generator produces **four types of JSON context files**, each describing a logical layer or group:

### 3.1 `workflow-context.json`
Describes detected end-to-end workflows inferred from naming correlation.

**Example**:
```json
[
  {
    "workflow_id": "LabelPrint",
    "components": [
      { "type": "UI", "file": "UI/Pages/LabelPrint.aspx", "path": "/repo/UI/Pages/LabelPrint.aspx" },
      { "type": "Service", "file": "Services/WCF/LabelPrintService.svc", "path": "/repo/Services/WCF/LabelPrintService.svc" },
      { "type": "DAL", "file": "DataAccess/Repositories/LabelRepository.cs", "path": "/repo/DataAccess/Repositories/LabelRepository.cs" }
    ],
    "detected_via": "naming_match",
    "last_updated": "2025-04-05T10:00:00Z"
  }
]
```

### 3.2 `service-context.json`
Describes all service endpoints and their types.

**Example**:
```json
[
  {
    "service_name": "LabelPrintService",
    "file": "Services/WCF/LabelPrintService.svc",
    "type": "WCF",
    "operations": ["PrintLabel", "ValidateTemplate"],
    "hosting_model": "IIS",
    "auth_required": true
  }
]
```

### 3.3 `dal-context.json`
Describes data access components and their SQL dependencies.

**Example**:
```json
[
  {
    "repository": "LabelRepository",
    "file": "DataAccess/Repositories/LabelRepository.cs",
    "mapped_sql_files": ["SQL/LabelSchema.sql", "SQL/Procedures/InsertLabel.sql"],
    "database": "InventoryDB"
  }
]
```

### 3.4 `ui-context.json`
Describes UI pages and associated client logic.

**Example**:
```json
[
  {
    "page": "LabelPrint.aspx",
    "path": "UI/Pages/LabelPrint.aspx",
    "scripts": ["Scripts/label-printer.js"],
    "backend_service": "LabelPrintService.svc"
  }
]
```

All output files are written to a configurable directory and are safe to commit to version control.

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
3. Determine **affected workflows** by:
   - Any file in a workflow group that changed → regenerate that workflow.
   - Service files → update `service-context.json`.
   - DAL-related files → update `dal-context.json` and any dependent workflows.
4. Only regenerate affected JSON files.
5. Update `state.json` with new timestamps and hashes.

### Options:
- `--full-scan`: Ignore state, regenerate all.
- `--incremental` (default): Use state for delta processing.

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