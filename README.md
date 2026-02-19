# TD Text-to-SQL Analytics Chat (Prototype) — PRD

## 1) Product name
**TD Text-to-SQL Analytics Chat (Prototype)**

## 2) Problem statement
TD users need a fast way to ask business questions about approved datasets without learning schemas, joins, or SQL. The system must stay read-only, avoid large data dumps, and remain secure and explainable.

## 3) Goals
- Provide a **chat-first** interface that answers data questions using **metadata-grounded SQL**.
- Support **multi-step clarification** when questions are ambiguous (up to **5** clarification turns).
- Support **analytics/report intent** including **charts** derived from aggregated query results.
- Be **modular**, **secure**, and **switchable** between **SQL Server (Azure SQL)** and **SQLite** for testing.
- Provide **debug visibility** (logs, retrieved metadata, prompts, SQL, tool calls, errors) when enabled.

## 4) Non-goals (v1)
- No write/update capability (DML/DDL) in any environment.
- No UI workflow for metadata upload/index rebuild (script/module only in v1).
- No image-based relationship extraction (relationships are text-defined in Excel).

## 5) Users & personas
- **Analyst / Business user**: asks questions, wants clear answers and charts.
- **Power user / Data user**: wants SQL transparency (debug mode), validation steps.
- **Developer / Support**: needs robust logs and reproducible error traces.

## 6) UX requirements

### 6.1 UI design
- **TD theme**: green + white, clean cards, logo placeholder OK.
- **Chat interface** as primary interaction model.
- Results shown as:
  - concise answer summary
  - small table preview (limited)
  - optional charts for analytics requests

### 6.2 UI options (must be configurable in UI)
These must be available as UI controls (top bar or settings panel):

1) **Debug mode (on/off)**
- Off: no internal details visible.
- On: inline debug panels per interaction:
  - intent classification decision
  - requirement-clarity decision + clarifying questions
  - retrieved metadata snippets + doc IDs (citations)
  - prompts sent to LLM (or safe summaries if required)
  - generated SQL (both dialects if produced)
  - execution timing + row/column counts
  - tool calls
  - handled/unhandled errors (stack traces optional)

2) **Max rows returned to UI** (default: 50, adjustable)
3) **Max columns returned to UI** (default: 20, adjustable)
4) **Max execution time** (default: e.g., 20s, adjustable)
5) **DB backend selector** (SQL Server / SQLite) — if allowed by environment

## 7) Functional requirements

### 7.1 Metadata ingestion & indexing (module/script)
- Input: `rrdw_meta_data.xlsx` with 3 sheets:
  - `field` (column metadata)
  - `table` (table business definitions)
  - `relationship` (join definitions in text)
- The ingestion module must:
  - validate schema
  - build three document sets
  - **drop and recreate** the Azure AI Search indexes (full refresh)
  - upload documents
- This does not need to be UI-driven for v1.

### 7.2 Retrieval (Azure AI Search)
- On each question:
  - query AI Search to retrieve top relevant metadata docs across:
    - field docs
    - table docs
    - relationship docs
- Provide citations (doc IDs + snippets) to downstream agents and optionally to UI (debug mode).

### 7.3 Agentic orchestration (AutoGen-based)
The system must include these layers/agents:

1) **Intent Router Agent**
- Categorizes request into:
  - Data Q&A (SQL)
  - Analytical report (SQL + charts)
  - General question (non-data)
  - Irrelevant/out-of-scope → redirect politely

2) **Requirement Clarity Agent**
- Checks if the request is answerable:
  - identifies missing constraints (date range, entity definition, domain meanings, etc.)
  - asks clarifying questions (up to 5 turns)
  - suggests choices when ambiguous
- Uses retrieval tools to ground clarifications.

3) **SQL Planner/Generator Agent**
- Generates SQL grounded in retrieved metadata + relationships.
- Produces both:
  - **SQL Server dialect** (primary target)
  - **SQLite dialect** (optional/testing)

4) **SQL Safety/Policy Layer (enforced)**
- Validates generated SQL:
  - SELECT-only
  - blocks UPDATE/INSERT/DELETE/MERGE/DDL
  - row/column limits enforced
  - query timeout enforced
  - prevents “return everything” behavior
- If unsafe → reject and ask user to reframe.

5) **Execution Layer**
- Executes SQL on the selected backend:
  - SQL Server (Azure SQL) primary
  - SQLite optional
- Returns only limited result sets (per UI settings).

6) **Result Interpretation Agent**
- Receives:
  - user question
  - SQL executed
  - limited result preview
- Produces:
  - business-friendly narrative
  - key numbers
  - caveats/assumptions
  - suggested follow-up questions

7) **Chart/Analytics Agent**
- For analytical report intent:
  - decides chart type(s)
  - generates charts from aggregated result sets
  - returns charts to UI
- Must not require loading huge raw data.

### 7.4 Error handling & robustness
- All unhandled errors must be:
  - visible in debug mode
  - logged to file
- When SQL execution fails:
  - the system provides the error context to the agentic layer
  - model decides whether to retry and how (repair SQL, ask clarifying Q, or stop)

### 7.5 Caching (Redis optional)
If enabled, Redis may be used for:
- session conversation state
- caching retrieval results
- caching query plans / successful SQL templates
- rate limiting / debouncing repeated queries

Redis must be optional and the system must run without it.

## 8) Performance / large tables requirements
- Some tables are huge:
  - avoid full table scans when possible
  - enforce max rows/cols/timeouts
  - prefer **aggregation-first** queries
  - do not load full datasets into UI
- Default “send results to LLM” row cap = **50**, adjustable.

## 9) Modularity requirements
- Modular architecture:
  - clear separation: retrieval, routing, clarity, SQL gen, safety, execution, interpretation, visualization
  - DB backend interchangeable through an interface
  - indexing module independent from runtime chat module

## 10) Security requirements
- **Managed Identity (MSI)** authentication for Azure OpenAI + Azure AI Search + Azure SQL.
- No secrets in code; config via environment variables.
- Enforce read-only analysis:
  - SELECT-only
  - denylist DDL/DML keywords
  - optional allowlist of schemas/tables/views (recommended)
- Logging must support debug output without exposing excessive data to end users (debug is optional and controlled).

---

# Non-Functional Requirements (NFR)

## Security
- MSI-based auth everywhere (no keys in code).
- Least privilege access.
- SQL enforcement: SELECT-only, no destructive operations, timeouts.
- Safe logging practices and the ability to redact if needed.

## Performance
- Designed for large tables: aggregation-first, row/column limits, timeouts.
- Caching optional via Redis.

## Reliability
- Graceful failure with actionable messages.
- Deterministic indexing refresh (drop + recreate).
- Minimal external dependencies (no Hugging Face).

## Maintainability
- Modular, unit-testable services/agents.
- Clear interfaces between steps.

## Observability
- Structured logs with correlation IDs.
- Inline debug panel in UI when enabled.

---

# Acceptance Criteria

## A) UI & UX
1) UI renders with **TD green/white theme**, clean cards, chat interface.
2) Debug mode OFF → user sees only final response + limited preview.
3) Debug mode ON → inline panels show:
   - intent decision
   - clarity checks + clarifying questions
   - retrieved metadata + citations
   - prompts (or safe summaries)
   - SQL generated (both dialects if applicable)
   - execution metrics (rows/cols/time)
   - handled/unhandled errors

## B) Metadata indexing
4) Running the indexing module:
   - validates Excel schema
   - drops and recreates 3 AI Search indexes
   - uploads docs successfully
5) Retrieval returns relevant field/table/relationship docs for a sample query.

## C) Safety
6) Any attempt to UPDATE/DELETE/INSERT/MERGE/DDL is blocked with a clear message.
7) Results always respect UI-configured max rows/cols/timeouts and never dump full datasets.

## D) Agentic behavior
8) Ambiguous questions trigger clarifying questions first (up to 5 turns).
9) Intent router correctly routes:
   - data Q&A → SQL path
   - analytics/report → SQL + chart path
   - general questions → non-data answer
   - irrelevant → redirect guidance

## E) Execution + interpretation
10) Valid data question returns:
   - narrative answer
   - limited result preview
11) Analytics request returns at least one chart based on aggregated results.

## F) DB switching
12) Switching backend from SQLite → Azure SQL requires no UI changes and minimal config changes.

## G) Logging
13) All errors are logged to file and visible in debug mode; logs are easy to copy into GitHub Copilot.
