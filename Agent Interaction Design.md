#  Agent Interaction Design

This document explains how the agents interact in the AutoGen-style, tool-driven architecture for the TD Text-to-SQL Analytics Chat system (read-only analytics, clarify-first, safe SQL, limited results, optional charts, TD-themed chat UI with debug panels).

---

## 1) Agent roles (minimum viable set)

### 1. Orchestrator (Supervisor)
**Responsibility**
- Owns conversation state and the end-to-end control flow.
- Enforces policies: *clarify-first*, max 5 clarification turns, read-only.
- Emits debug logs and collects step outputs for inline UI panels.

**Inputs**
- User message, session state, UI settings (debug/max rows/max cols/timeout/backend)

**Outputs**
- Final response to UI + debug payload

---

### 2. Intent Router Agent
**Responsibility**
- Classifies the user request into one of:
  - `DATA_QA` (needs SQL + answer)
  - `ANALYTICS_REPORT` (needs SQL + charts + answer)
  - `GENERAL_QA` (non-data or metadata-only)
  - `OUT_OF_SCOPE` (redirect politely)

**Outputs**
- `intent`, `confidence`, `reason`, `suggested_next_agent`

---

### 3. Requirement Clarity Agent
**Responsibility**
- Checks if the user request is clear enough to proceed.
- If unclear, asks clarifying questions and suggests options.
- Uses metadata tools to avoid hallucinations.

**Rules**
- Ask before SQL generation if ambiguous.
- Max clarification loops: **5**.

**Outputs**
- `is_clear`
- `clarifying_questions[]`
- `assumptions_if_proceeding[]`

---

### 4. Metadata Retriever Agent
**Responsibility**
- Retrieves top relevant metadata from Azure AI Search:
  - table definitions
  - field definitions
  - relationship/join rules

**Outputs**
- `grounding_pack` containing:
  - `table_docs[]`, `field_docs[]`, `relationship_docs[]`
  - `citations[]` (doc IDs + snippets)
  - `suggested_tables[]`
  - `suggested_join_paths[]`

---

### 5. SQL Generator Agent
**Responsibility**
- Generates SQL grounded in retrieved metadata + relationships.

**Dialect requirement**
- Produces **two SQL variants**:
  - `sql_server` (primary target)
  - `sql_sqlite` (secondary/testing)

**Outputs**
- `sql_server`, `sql_sqlite`
- `used_tables[]`
- `notes`, `assumptions[]`

---

### 6. SQL Safety Guard Agent (enforced)
**Responsibility**
- Validates SQL is safe and read-only:
  - SELECT-only
  - block DDL/DML (UPDATE/INSERT/DELETE/MERGE/DROP/ALTER/CREATE/TRUNCATE)
  - enforce max rows/max cols/timeouts
  - avoid dumping huge tables (aggregation-first)
- May rewrite to add TOP/LIMIT, or reject with a user-facing message.

**Outputs**
- `is_safe`
- `safe_sql_server`, `safe_sql_sqlite` (possibly rewritten)
- `violations[]` (if unsafe)
- `user_message` (if blocked)

---

### 7. DB Executor Agent
**Responsibility**
- Executes SQL against configured backend:
  - SQL Server (Azure SQL)
  - SQLite (optional/testing)
- Returns limited preview + stats.

**Outputs**
- `result_preview` (columns + rows, truncated)
- `execution_stats` (elapsed time, rows returned, truncated bool)
- `db_error` (if any)

---

### 8. Result Interpreter Agent
**Responsibility**
- Turns query results into a business-friendly answer.
- Uses only the limited preview and execution stats.

**Outputs**
- `answer`
- `highlights[]` (optional)
- `followups[]` (suggested next questions)

---

### 9. Chart Builder Agent (for analytics intent)
**Responsibility**
- Generates charts from aggregated results (never full raw dumps).
- Produces chart specs and/or rendered artifacts for UI.

**Outputs**
- `chart_specs[]` (type, x/y columns, title)
- `charts[]` (paths/bytes depending on UI framework)

---

### 10. Error Triage Agent
**Responsibility**
- On SQL execution errors, decides next action:
  - patch SQL and retry
  - ask clarifying question
  - stop and explain

**Your requirement**
- “Let GPT decide to retry based on the error message.”

**Outputs**
- `action` in {`RETRY_WITH_PATCH`, `ASK_CLARIFICATION`, `STOP`}
- `patched_sql_server` / `patched_sql_sqlite` (if retry)
- `clarifying_questions[]` (if ask)
- `user_message` (if stop)

---

## 2) Primary interaction flows

### A) Data Q&A (SQL + answer)
1. **User → Orchestrator**: message
2. **Orchestrator → Intent Router**
3. **Intent Router → Orchestrator**: `DATA_QA`
4. **Orchestrator → Requirement Clarity**
5. If unclear: **Orchestrator → UI** asks clarifying questions (repeat, max 5)
6. **Orchestrator → Metadata Retriever** (Azure AI Search)
7. **Metadata Retriever → Orchestrator**: grounding pack + citations
8. **Orchestrator → SQL Generator**
9. **SQL Generator → Orchestrator**: SQL Server + SQLite SQL
10. **Orchestrator → SQL Safety Guard**
11. **SQL Safety Guard → Orchestrator**: safe/blocked + rewritten SQL if needed
12. **Orchestrator → DB Executor**
13. **DB Executor → Orchestrator**: preview results + stats (or error)
14. **Orchestrator → Result Interpreter**
15. **Result Interpreter → Orchestrator**: narrative answer + follow-ups
16. **Orchestrator → UI**: answer + limited table preview

**Debug mode (inline panels)** shows each step output.

---

### B) Analytics report (SQL + charts + answer)
Same as Data Q&A through step 15, then:
16. **Orchestrator → Chart Builder**
17. **Chart Builder → Orchestrator**: chart(s)
18. **Orchestrator → UI**: answer + chart(s) + limited preview

---

### C) Ambiguous question (clarify-first)
Example: “How is my dollar value shifting from yesterday to today?”
- Requirement Clarity Agent asks:
  - Which metric? (balance, sum of transactions, etc.)
  - Which product group? (deposits vs savings)
  - What date boundary/time zone?
- Only after clarifications does the flow proceed to retrieval → SQL → execution.

---

### D) SQL error flow (model decides retry)
1. DB Executor returns `db_error`.
2. Orchestrator calls **Error Triage Agent** with:
   - the SQL attempted
   - the DB error message
   - grounding pack citations
3. Error Triage decides:
   - retry with patch
   - ask clarification
   - stop
4. Orchestrator executes the decision and logs it.

---

## 3) Tools required (map directly to AutoGen tools)

### Core tools
- `search_metadata(query, top_k, filters)` → Azure AI Search
- `llm_call(prompt, model="gpt-4.1")` → Azure OpenAI (MSI auth)
- `execute_sql(sql, backend, timeout, max_rows, max_cols)` → DB layer
- `render_chart(data, chart_spec)` → local matplotlib/plotly
- `log_event(step, payload)` → file + in-memory buffer

### Notes
- Avoid external internet/HuggingFace dependencies.
- Redis is optional for caching sessions, retrieval results, or deduping.

---

## 4) Orchestrator state (stored per chat turn)
- `user_question`
- `intent`
- `clarification_turn_count`
- `clarifying_questions_asked[]`
- `grounding_pack` (docs + citations)
- `sql_server`, `sql_sqlite`
- `safety_decision`
- `execution_stats` (time, rows, truncated)
- `result_preview` (limited rows/cols)
- `final_answer`
- `charts` (optional)
- `errors` (if any)

This state powers debug panels and reproducibility for GitHub Copilot troubleshooting.

---

## 5) Recommended initial wiring (fast + deterministic)
- Start with a **single supervisor Orchestrator** + specialist agents called in a deterministic pipeline.
- Avoid full “free-form multi-agent group chat” initially; add later if needed.
- Keep retries controlled and auditable.

---

## 6) Acceptance checklist for agent interaction
- Intent routing works and is logged.
- Clarify-first enforced, max 5 turns.
- Retrieval citations are visible in debug.
- SQL generated in two dialects (SQL Server + SQLite).
- Safety guard blocks non-SELECT and enforces limits.
- Execution returns limited preview only.
- Interpretation returns user-friendly narrative.
- Analytics intent produces charts from aggregates only.
- Errors are logged; Error Triage decides retry behavior.
