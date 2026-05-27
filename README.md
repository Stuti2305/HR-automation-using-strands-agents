# 🏢 HR Multi-Agent System

An AI-powered HR assistant built with the **Strands Agents SDK** and **Ollama**, featuring a Supervisor Agent that intelligently routes employee queries to specialized sub-agents. Observability is provided via **LangSmith**.

---

## Overview

Employees interact with a single entry point — the Supervisor Agent — which classifies each query and delegates it to the right specialist:

| Query Type | Routed To |
|---|---|
| WFH policy, benefits, leave rules | Policy Agent |
| Apply leave, check leave status | Leave Agent |
| First-day tasks, onboarding steps | Onboarding Agent |

---

## Features

- **Multi-agent routing** — Supervisor uses Strands tools to delegate, not hardcoded logic
- **Leave management** — Apply and check leave requests with in-memory persistence and auto-generated request IDs
- **Policy Q&A** — Answers questions about WFH, benefits, PF, health insurance, and learning budget
- **Onboarding guidance** — Step-by-step first-day and first-week instructions for new joiners
- **LangSmith observability** — Traces, metrics, and a dashboard for every request
- **Local LLM** — Runs fully offline using Ollama (`qwen2.5:0.5b`)

---

## Project Structure

```
HRAGENT-F.ipynb          # Main notebook (all cells in sequence)
hr_observability_traces.jsonl   # Auto-generated trace log (runtime)
README.md
ARCHITECTURE.md
requirements.txt
```

---

## Quick Start

### Prerequisites

- Python 3.9+
- [Ollama](https://ollama.com) installed and running locally
- `qwen2.5:0.5b` model pulled

```bash
ollama pull qwen2.5:0.5b
ollama serve          # keep this running in a separate terminal
```

### Install dependencies

```bash
pip install strands-agents[ollama] strands-agents-tools langsmith
```

Or use the requirements file:

```bash
pip install -r requirements.txt
```

### Run

Open `HRAGENT-F.ipynb` in Jupyter and run all cells top to bottom (Cell 1 → Cell 8), then run any test query block.

---

## Usage

All queries go through the `ask()` helper:

```python
# Policy query
ask("What is the work from home policy at HCL Corp?")

# Leave application
ask("Apply leave for John Doe from 2025-12-20 to 2025-12-27 for family vacation")

# Onboarding
ask("Hi! I just joined HCL Corp today. What should I do first?")
```

Check the leave database directly:

```python
for item in LEAVE_DB.values():
    print(item)
```

View the observability dashboard:

```python
show_observability_dashboard()
show_recent_traces()
```

---

## LangSmith Observability

Set your LangSmith API key in Cell 3 before running. Traces are viewable at:

```
https://smith.langchain.com/  →  Project: HR-MultiAgent-System
```

A local JSONL trace file (`hr_observability_traces.jsonl`) is also written for offline inspection.

---

## HR Policies Covered

| Policy | Detail |
|---|---|
| WFH | Up to 3 days/week, manager approval required; not in first 3 months |
| Annual Leave | 21 days/year |
| Sick Leave | 10 days/year |
| Health Insurance | Employee + family covered |
| Learning Budget | INR 25,000/year |
| PF | 12% of basic salary |

---

## Tech Stack

| Component | Technology |
|---|---|
| Agent Framework | Strands Agents SDK |
| LLM | Ollama — `qwen2.5:0.5b` |
| Observability | LangSmith + local JSONL |
| Runtime | Jupyter Notebook (Python 3.9+) |
| Leave Storage | In-memory Python dict |

---

## Notes

- `LEAVE_DB` is in-memory and resets when the kernel restarts. For persistence, replace it with a database or file-based store.
- The Leave Agent uses regex-based parsing instead of LLM tool-calling for reliability with small models.
- The Supervisor Agent requires the `qwen2.5:0.5b` model to reliably call tools; larger models will improve routing accuracy.
