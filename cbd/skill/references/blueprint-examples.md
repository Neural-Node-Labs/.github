# CBD Blueprint Examples

Reference blueprints for common system types. Use these as calibration for your own output.

---

## Example 1: REST API with LLM Integration

**System**: User submits a support ticket → AI categorizes it → routed to correct team queue.

### Master Blueprint Table

| # | Component Name | Layer | Reason for Existence | IN-Schema | OUT-Schema | Error-Schema | Adaptor Needed | Trace Points | Risk |
|---|---|---|---|---|---|---|---|---|---|
| 1 | `Logger` | Foundation | Emit structured log entries to both sinks | `{level: enum[INFO,WARN,ERROR], message: string, component: string, metadata: object}` | `{logged: bool, timestamp: ISO8601}` | `{code: "LOG_FAIL", message: string, component: "Logger", timestamp: ISO8601}` | None | Write confirmation | LOW |
| 2 | `ConfigLoader` | Foundation | Load and expose validated env/config values at startup | `{env: string}` | `{llmApiKey: string, queueUrl: string, maxRetries: int}` | `{code: "CONFIG_MISSING", message: string, component: "ConfigLoader", timestamp: ISO8601}` | None | Keys loaded (values redacted) | LOW |
| 3 | `SchemaValidator` | Foundation | Validate any payload against a declared schema contract | `{payload: object, schema: JSONSchema}` | `{valid: bool, errors: string[]}` | `{code: "VALIDATION_ERROR", message: string, component: "SchemaValidator", timestamp: ISO8601}` | None | Field count, pass/fail | LOW |
| 4 | `TicketIngestionAdaptor` | Adaptor | Map raw HTTP request body to TicketProcessor IN-Schema | `{httpBody: object, headers: object}` | `{ticketId: uuid, userText: string, userId: string, submittedAt: ISO8601}` | `{code: "INGEST_MAP_FAIL", ...}` | None | Fields mapped, fields dropped | MEDIUM |
| 5 | `TicketProcessor` | Domain | Sanitize and enrich raw ticket data before AI processing | `{ticketId: uuid, userText: string, userId: string, submittedAt: ISO8601}` | `{ticketId: uuid, cleanText: string, charCount: int, language: string}` | `{code: "PROCESS_FAIL", ...}` | TicketIngestionAdaptor | Input length, language detected | LOW |
| 6 | `LLMCategorizerAdaptor` | Adaptor | Translate TicketProcessor output to LLM prompt format | `{ticketId: uuid, cleanText: string}` | `{prompt: string, systemMessage: string, ticketId: uuid}` | `{code: "PROMPT_BUILD_FAIL", ...}` | None | Prompt token estimate | MEDIUM |
| 7 | `LLMCategorizer` | Domain | Call LLM API and extract category from response | `{prompt: string, systemMessage: string, ticketId: uuid}` | `{ticketId: uuid, category: enum[billing,technical,general], confidence: float}` | `{code: "LLM_FAIL", ...}` | LLMCategorizerAdaptor | Raw prompt, raw response, category extracted | CRITICAL |
| 8 | `QueueRouterAdaptor` | Adaptor | Map category to correct queue endpoint URL | `{ticketId: uuid, category: string}` | `{ticketId: uuid, queueUrl: string, priority: int}` | `{code: "ROUTE_MAP_FAIL", ...}` | None | Category→URL mapping used | LOW |
| 9 | `QueueDispatcher` | Domain | POST ticket to resolved queue endpoint | `{ticketId: uuid, queueUrl: string, priority: int}` | `{ticketId: uuid, dispatched: bool, queueAck: string}` | `{code: "DISPATCH_FAIL", ...}` | QueueRouterAdaptor | HTTP status, retry count | HIGH |
| 10 | `TicketOrchestrator` | Orchestrator | Sequence all components for a single ticket submission | `{httpRequest: object}` | `{ticketId: uuid, status: enum[routed,failed], summary: string}` | `{code: "ORCHESTRATION_FAIL", ...}` | None | Phase transitions, total latency | MEDIUM |

### Technical Risk Assessment

**Critical Bottlenecks**
- `LLMCategorizer` (CRITICAL): Non-deterministic LLM output may return categories outside the defined enum. Implement strict output parsing with fallback to `general`. Circuit breaker: 3 failures → default all to `general` + alert.

**Integration Friction Points**
- `LLMCategorizerAdaptor` → `LLMCategorizer`: Token limits may be breached for long tickets. Adaptor must truncate `cleanText` to max 2000 chars with a trace log warning.

**Stability Warnings**
- `QueueDispatcher`: External queue may be slow or down during peak load. Exponential backoff required; retry count must be capped at `ConfigLoader.maxRetries`.

**Feasibility Verdict**: **GO** — All dependencies are standard HTTP + LLM API calls. Highest risk is LLM non-determinism, which is mitigated by enum enforcement in the adaptor layer.

---

## Example 2: ETL Data Pipeline

**System**: Read CSV files → transform → validate → load to database.

### Master Blueprint Table (abbreviated)

| # | Component Name | Layer | Reason for Existence | IN-Schema | OUT-Schema | Risk |
|---|---|---|---|---|---|---|
| 1 | `Logger` | Foundation | Dual-stream structured logging | `{level, message, component, metadata}` | `{logged: bool, timestamp}` | LOW |
| 2 | `ConfigLoader` | Foundation | Load DB credentials and file paths | `{env: string}` | `{dbConnStr: string, inputPath: string, batchSize: int}` | LOW |
| 3 | `FileScanner` | Domain | Discover all unprocessed CSV files in input directory | `{inputPath: string, processedLog: string[]}` | `{files: [{path: string, sizeBytes: int, discoveredAt: ISO8601}]}` | LOW |
| 4 | `CSVParser` | Domain | Parse a single CSV file into typed row records | `{filePath: string}` | `{rows: [{...typed fields}], rowCount: int, parseErrors: int}` | MEDIUM |
| 5 | `CSV_To_Domain_Adaptor` | Adaptor | Map CSV column names to domain model field names | `{rows: object[]}` | `{records: DomainRecord[]}` | MEDIUM |
| 6 | `RecordValidator` | Domain | Validate each domain record against business rules | `{records: DomainRecord[]}` | `{valid: DomainRecord[], invalid: [{record, reason}]}` | MEDIUM |
| 7 | `DatabaseWriter` | Domain | Batch insert valid records into target DB table | `{records: DomainRecord[], batchSize: int}` | `{inserted: int, failed: int, duration_ms: int}` | HIGH |
| 8 | `ETLOrchestrator` | Orchestrator | Sequence scan→parse→adapt→validate→write per file | `{env: string}` | `{filesProcessed: int, totalInserted: int, errors: ErrorSchema[]}` | MEDIUM |

---

## Example 3: CLI Tool

**System**: Developer runs `analyze-repo --path ./src` → produces code quality report.

**Key design note**: CLI tools often skip Adaptors if all data stays in-process. Only add Adaptors when integrating external APIs or when two domain components have mismatched schemas.

### Master Blueprint Table (abbreviated)

| # | Component Name | Layer | Reason for Existence | IN-Schema | OUT-Schema | Risk |
|---|---|---|---|---|---|---|
| 1 | `CLIArgumentParser` | Foundation | Parse and validate CLI flags into typed config | `{argv: string[]}` | `{path: string, outputFormat: enum[text,json], verbose: bool}` | LOW |
| 2 | `FileTreeWalker` | Domain | Recursively list all source files matching filter | `{rootPath: string, extensions: string[]}` | `{files: [{path: string, extension: string, sizeBytes: int}]}` | LOW |
| 3 | `FileAnalyzer` | Domain | Compute complexity metrics for a single file | `{filePath: string}` | `{path: string, lineCount: int, cyclomaticComplexity: int, issues: string[]}` | MEDIUM |
| 4 | `ReportAggregator` | Domain | Combine per-file results into a summary report | `{analyses: FileAnalysis[]}` | `{totalFiles: int, avgComplexity: float, topIssues: string[], score: int}` | LOW |
| 5 | `ReportRenderer` | Domain | Format report as text or JSON for terminal output | `{report: AggregatedReport, format: enum[text,json]}` | `{output: string}` | LOW |
| 6 | `CLIOrchestrator` | Orchestrator | Sequence all components for a single CLI invocation | `{argv: string[]}` | `{exitCode: int}` | LOW |
