# 🏗️ Architecture — HR Multi-Agent System

---

## High-Level Flow

```
Employee Query
      │
      ▼
┌─────────────────────┐
│   ask() Helper      │  ← Entry point; creates TraceContext, calls supervisor
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Supervisor Agent   │  ← Strands Agent with 3 routing tools
│  (qwen2.5:0.5b)     │
└────────┬────────────┘
         │  routes via tool call
    ┌────┴────────────────────┐
    │            │            │
    ▼            ▼            ▼
┌────────┐ ┌─────────┐ ┌────────────┐
│ Policy │ │  Leave  │ │ Onboarding │
│ Agent  │ │  Agent  │ │   Agent    │
└────────┘ └────┬────┘ └────────────┘
                │
       ┌────────┴────────┐
       ▼                 ▼
 apply_leave()     check_leave()
  (@tool)           (@tool)
       │
       ▼
   LEAVE_DB
 (in-memory dict)
```

---

## Component Details

### Supervisor Agent

- **Type:** Strands `Agent` with tools
- **Model:** `qwen2.5:0.5b` via Ollama
- **Tools registered:** `ask_policy_tool`, `ask_leave_tool`, `ask_onboarding_tool`
- **Routing logic:** The system prompt defines three routing rules. The LLM selects the correct tool based on the user's intent and passes the original query to the sub-agent.
- **Fallback:** If the LLM response is too short or contains raw tool-call text, `ask()` walks `supervisor.messages` to extract the actual tool result.

---

### Policy Agent

- **Type:** Strands `Agent` (no tools)
- **Model:** `qwen2.5:0.5b` via Ollama
- **Behaviour:** Pure Q&A. System prompt embeds all policy data directly so the LLM answers without external lookups.
- **Covers:** WFH rules, annual/sick leave entitlements, health insurance, learning budget, PF contribution.

---

### Leave Agent

- **Type:** Custom `LeaveAgentWrapper` class (not a Strands Agent)
- **Design rationale:** Small LLMs are unreliable at structured tool-calling for leave data extraction. Regex parsing on employee name, dates (`YYYY-MM-DD`), and reason is more robust.
- **Routing within Leave Agent:**
  - Contains keyword `"check"` + `"leave"` → calls `check_leave_status()`
  - Otherwise → calls `handle_leave_request()` → `apply_leave()` tool

---

### Onboarding Agent

- **Type:** Strands `Agent` (no tools)
- **Model:** `qwen2.5:0.5b` via Ollama
- **Behaviour:** Answers new-joiner questions using a structured Day 1 checklist and Week 1 action items embedded in the system prompt.

---

## Tools

### `apply_leave` (`@tool`)

```
Inputs : employee_name, start_date (YYYY-MM-DD), end_date (YYYY-MM-DD), reason
Output : JSON — { success, request_id, message }
Side effect : Writes to LEAVE_DB (in-memory dict)
Also logs : LangSmith run via log_to_langsmith()
           Metrics via update_metrics(tool="apply_leave")
```

### `check_leave` (`@tool`)

```
Inputs : employee_name
Output : JSON — { employee, total, requests[] } or { message }
Source : Queries LEAVE_DB by case-insensitive name match
Also logs : LangSmith + metrics
```

---

## Observability Layer

```
Every ask() call
      │
      ├─► TraceContext created (UUID, start_time, events list)
      │
      ├─► Events appended at each step (user_input, routing, response, error)
      │
      ├─► update_metrics() — increments METRICS_STORE counters
      │       total_requests, total_tool_calls, agent_requests{}, tool_calls{}
      │
      ├─► log_to_langsmith() — POSTs a run dict to LangSmith via batch_ingest_runs()
      │       run_type: chain / tool, tags: ["hr-multiagent"]
      │
      └─► trace.save_trace() — appends all events to hr_observability_traces.jsonl
```

### Dashboard functions

| Function | Output |
|---|---|
| `show_observability_dashboard()` | Prints totals, per-agent counts, per-tool counts, LangSmith link |
| `show_recent_traces(limit=5)` | Reads JSONL, prints last N traces with status and input preview |

---

## Data Flow — Leave Application Example

```
User: "Apply leave for Priya Sharma from 2026-01-05 to 2026-01-10 for vacation"
  │
  ▼
ask() → supervisor("Apply leave for Priya Sharma ...")
  │
  ▼  [Supervisor LLM selects ask_leave_tool]
ask_leave_agent("Apply leave for Priya Sharma ...")
  │
  ▼  update_metrics(agent="leave")
LeaveAgentWrapper.__call__()
  │
  ▼  regex extracts: name="Priya Sharma", start="2026-01-05", end="2026-01-10", reason="vacation"
handle_leave_request()
  │
  ▼
apply_leave("Priya Sharma", "2026-01-05", "2026-01-10", "vacation")
  │
  ├─► LEAVE_DB["LV-XXXXXX"] = { request_id, employee, dates, reason, status="Pending Approval" }
  ├─► update_metrics(tool="apply_leave")
  └─► log_to_langsmith("tool", "apply_leave", inputs, outputs)
  │
  ▼
"✅ Leave applied! Request ID: LV-XXXXXX. Status: Pending Approval."
```

---

## Cell Execution Order

| Cell | Purpose |
|---|---|
| 1 | Install `strands-agents[ollama]`, `strands-agents-tools`, `langsmith` |
| 2 | Connect to Ollama, validate model with a test agent |
| 3 | Configure LangSmith env vars, initialise `METRICS_STORE` |
| 4 | Define `TraceContext`, `update_metrics()`, `log_to_langsmith()`, dashboard functions |
| 5 | Define `apply_leave` and `check_leave` tools, initialise `LEAVE_DB` |
| 6 | Instantiate Policy Agent, Leave Agent (wrapper), Onboarding Agent |
| 7 | Instantiate Supervisor Agent with the three routing tools |
| 8 | Define `ask()` helper |
| 9+ | Test queries and observability output |

---

## Key Design Decisions

**Why Ollama + qwen2.5:0.5b?**
Fully local, no API cost, fast on CPU. The 0.5B model is sufficient for classification and policy Q&A; the Leave Agent bypasses the LLM entirely for structured data extraction.

**Why a custom LeaveAgentWrapper instead of a Strands Agent?**
Small LLMs frequently fail to call structured tools with correct arguments. Deterministic regex parsing on the Leave Agent eliminates this failure mode without sacrificing functionality.

**Why in-memory LEAVE_DB?**
Keeps the notebook self-contained for demo purposes. Replace with SQLite, PostgreSQL, or a cloud store for production use.

**Why LangSmith + local JSONL?**
LangSmith provides a rich UI for trace inspection. The local JSONL ensures traces are never lost if the LangSmith call fails (errors in `log_to_langsmith` are silently swallowed to protect the main flow).
