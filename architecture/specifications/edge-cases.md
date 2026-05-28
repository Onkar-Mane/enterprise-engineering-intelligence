# Edge Cases for Enterprise Engineering Intelligence Platform  

This document outlines potential edge cases encountered by an enterprise engineering intelligence platform while indexing and reasoning over large ASP.NET enterprise codebases. Each edge case category includes a description, detection approach, platform behavior, and developer action required.

---

## 0. Edge Cases Already Handled by the Phase 1 Scanner (tree-sitter AST)

Several edge cases that were originally listed as risks are now handled deterministically by the Phase 1 scanner. They are included here for completeness — no developer action is required for these.

| Edge case | How it is handled | Validated against |
|---|---|---|
| **Commented-out methods being indexed as real** | tree-sitter AST ignores comments. Regex previously extracted 6 phantom methods from BLL files (218→212 after AST). | `MoldingBLL.cs`, `ManufactureBLL.cs` |
| **Non-public methods being missed** | tree-sitter walks all `method_declaration` nodes regardless of modifier. Regex previously required `public` keyword, missed 130 SVC methods (217→347, +37%). | `ManufactureService.svc.cs` |
| **Fully-qualified `[WebMethod]` attributes missed** | AST attribute node check with substring scan: `any("WebMethod" in attr for attr in m["attributes"])`. Catches both `[WebMethod]` and `[System.Web.Services.WebMethod]`. | `iPAS_SOPWebService` (was 0/229, now 229/229) |
| **Namespace-qualified ServiceClient breaks call detection** | Regex pattern accepts optional `[\w.]*\.` prefix on both sides, then asserts short-name match. `ManufactureBLL`: 479/603 → 603/603 service_calls. | `ManufactureBLL.cs` |
| **SQL string fragments masquerading as table names** | Guard: real tables are ALL_CAPS (`LDB1_*`, `TBL_*`) with min 4 chars. Drops fragments like `"Manage"`, `"one"`, `"SOP"` that bleed from SQL built in C# strings. | `MoldingDAL.cs`, `ManufactureDAL.cs` |
| **Conflicting method signatures across overloads** | Detected and flagged in per-file `conflicts` array with all line numbers; developer resolves in `cfg` field during Phase 2. | Per-file output |
| **Duplicate-named methods across BLL and SVC layers** | `service_calls` captures the actual ServiceClient method called, not the BLL wrapper. So `BLL.MarkManufactureCompleted → SVC.MarkManufactureForceCompleted` resolves correctly. | `ManufactureBLL.cs` |

**Magic numbers, business-logic intent, and undocumented status codes** are intentionally deferred to Phase 2 (LLM enrichment + developer Q&A) — these require semantic judgment the scanner cannot make. See edge case #1 below.


## 1. Cryptic or Undocumented Code  

### Description  
Code with no comments or documentation that makes reasoning difficult for the platform. Examples include:  
- Magic numbers (`4`, `5`, `0`,`-1`) without context or comments.  
- Custom status codes or error codes only understood by one developer.  
- Generic method names that offer no insight into functionality (e.g., `Process()`, `Handle()`, `Execute()`).

### Detection Approach  
- **Static Analysis**: Detect usage of hardcoded numerical values without accompanying comments or explanations.  
- **Pattern Matching**: Flag generic method names and assess their usage across various contexts.  
- **Heuristics**: Identify methods where high interaction exists but low semantic meaning is inferred from metadata.  

### Platform Behavior  
- Index flagged entities with metadata linking them to ambiguous contexts.  
- Generate warnings or notes in the platform's UI for developers to review and add documentation.  
- Suggest semantic improvements by identifying frequently accessed or used cryptic methods.  

### Developer Action Required  
- Review flagged instances and provide documentation for non-descriptive methods, magic numbers, and hardcoded status codes.  
- Refactor ambiguous method names to accurately convey their purpose.  
- Replace magic numbers with named constants or enums accompanied by inline comments.



## 2. Dynamic Runtime Behavior  

### Description  
Code whose behavior changes based on runtime conditions such as:  
- Session state or configuration values.  
- Reflection-based method calls (e.g., `methodInfo.Invoke()`), making static analysis difficult.  
- Dependencies injected dynamically in runtime, creating non-deterministic logic chains.

### Detection Approach  
- **Session Hooks**: Identify methods that leverage session variables or global state.  
- **Reflection Analysis**: Flag usage of reflection APIs such as `Type.GetMethod()` or `Activator.CreateInstance()`.  
- **Dependency Tracing**: Attempt to trace dependency injection chains, with limitations for runtime-injected dependencies.  

### Platform Behavior  
- Warn about areas of the code that exhibit non-deterministic runtime behavior.  
- Partially index these areas with placeholder metadata and context notes.  
- Highlight skipped entities in the platform interface where runtime analysis is required.  

### Developer Action Required  
- Provide runtime scenarios or unit tests for flagged code paths.  
- Refactor overly dynamic code to make logic paths deterministic where possible.  
- Add comments or structures clarifying intended runtime behavior for index gaps.



## 3. Hardcoded Dependencies  

### Description  
Directly embedded dependencies in code create challenges during indexing or environment migrations. Examples include:  
- Connection strings hardcoded into `.cs` files.  
- Absolute or environment-specific paths/URLs directly embedded without config abstraction.

### Detection Approach  
- **String Literal Analysis**: Detect connection strings and absolute paths via regular expressions during indexing.  
- **Configuration Usage Audit**: Cross-check hardcoded values against external configuration files (e.g., Web.config, App.config).  

### Platform Behavior  
- Highlight entities with hardcoded dependencies in the platform UI for manual review.  
- Context generator attempts to associate these values with known configuration settings or environment mappings, but may fail for unknown references.  
- Raise recommendations for migrating hardcoded dependencies to configuration files or environment variables.  

### Developer Action Required  
- Refactor hardcoded dependencies into environment-agnostic configuration files or variables.  
- Use dependency injection frameworks or app settings to manage environment-specific data (e.g., connection strings, file paths).  
- Add fallback logic where environment values are unavailable.



## 4. Partial Outages During Indexing  

### Description  
Failures stemming from external or systemic issues during indexing. Examples of partial outages include:  
- Unavailable network share mid-scan.  
- File locked by IIS when accessing dynamically-generated data files.  
- Disk space exhaustion while writing intermediary or final results.

### Detection Approach  
- **Error Handling**: Monitor I/O exceptions, locked file statuses, and system disk health during indexing.  
- **Incremental State Management**: Maintain checkpoints of indexing progress to enable recovery.  

### Platform Behavior  
- Automatically pause indexing and retry failed operations after a fixed interval.  
- Skip unavailable files temporarily and log details for developer review.  
- Preserve partially completed indexing state and notify the developer of required intervention if recovery does not complete.  

### Developer Action Required  
- Resolve identified systemic issues (e.g., unlocking files, freeing disk space).  
- Restart indexing from the last saved checkpoint using the platform tooling.  
- Evaluate failure logs and ensure availability of required resources for future scans.



## 5. Conflicting Implementations Across Files  

### Description  
Duplicate or conflicting logic implemented differently across services or files. Examples include:  
- Same business logic appearing with subtle variations across two services.  
- Minor differences in syntax or parameters between copy-paste DAL methods.  

### Detection Approach  
- **Code Similarity Analysis**: Use semantic diff algorithms to detect subtle variations across similar blocks of code.  
- **Business Functionality Mapping**: Flag conflicting execution paths resulting from duplication across services.  

### Platform Behavior  
- Surface conflicting implementations in the platform UI with contextual differences outlined.  
- Generate conflict summaries highlighting signature differences, identical sections, and unique variations.  
- Offer recommendations for consolidation or annotation of reasoning where conflicts are intentional.  

### Developer Action Required  
- Inspect flagged conflicts and decide whether they require consolidation into reusable components.  
- Standardize business logic implementation across services to maintain consistency.  
- Annotate intentional differences in functionality to avoid future conflict flags.


