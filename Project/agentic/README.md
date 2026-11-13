# LangGraph Agent Workflow (`agentic/` + `tools/`)

> **In one sentence:** when an analyst asks for an assessment of an incident, a LangGraph workflow runs ten steps – tools compute the hard facts, five AI agents write the explanations and a recommendation – and then the workflow **pauses** until a human Data Protection Officer (DPO) decides what to do. Only after that decision does it draft a customer notice and/or a regulator filing.

---

## 1. Key ideas (read this first)

### 1.1 What is LangGraph?
LangGraph is a library for building workflows as a **graph**:
- **Nodes** = steps (plain Python functions).
- **Edges** = which step runs next (can be fixed, or chosen by a condition).
- **State** = a shared dictionary that flows through the graph. Each node reads what it needs and returns only the keys it adds.
- **Checkpointer** = saves the state after every step to a database, so a run can **pause and resume later** – even after the server restarts.
- **`interrupt()`** = a node can stop the graph and wait for outside input (here: a human decision).

### 1.2 "Tools compute, agents explain"
This is the most important design rule. Numbers that have legal consequences – harm probability, deadlines, hours left – are **never** produced by the language model. They are computed by deterministic Python tools (ML models + rules + facts read from the procedure PDFs). The agents are handed those facts and told:

> *"Every figure below was established by Merrowfield's models, documents and rules. Do not recalculate, round differently or change any figure, band, deadline or time; quote them as given."*

The agents' job is to **explain**, **connect the dots** and **write** – what LLMs are good at.

### 1.3 Human in the loop
The AI **recommends**; a person **decides**. Nothing is filed and no customer is told until a DPO records a decision with a written reason.

---

## 2. The graph

```
 START
   │
   ▼
 1 load_incident_context        tools only – profile, notes, campaign, harm & attack facts
   │
   ▼
 2 plan_assessment              🤖 agent 1: incident intake planner
   │
   ▼
 3 assess_harm                  🤖 agent 2: harm analyst
   │
   ▼
 4 identify_attack              🤖 agent 3: attack analyst
   │
   ▼
 5 check_reporting_duties       tools (procedures, deadlines) + 🤖 agent 4: reporting duty advisor
   │
   ▼
 6 compose_incident_assessment  tools (effort vs deadline) + 🤖 agent 5: incident writer
   │                            → saved as PENDING_DECISION
   ▼
 7 request_dpo_decision         ⏸  interrupt() – the graph STOPS here and is checkpointed
   │
   │   … minutes / hours later the DPO decides in the dashboard …
   │
   ├── "RECORD ONLY" ─────────────────────────────────────────────┐
   │                                                              │
   ├── "NOTIFY REGULATOR" ───────────────────┐                    │
   │                                         ▼                    │
   └── "NOTIFY REGULATOR AND CUSTOMERS"  9 draft_regulator_filing │  🤖 agent 7
             │                               ▲                    │
             ▼                               │                    │
         8 draft_customer_notice ────────────┘                    │
             🤖 agent 6                                           │
                                             │                    │
                                             ▼                    ▼
                                      10 record_incident_outcome  (tools only – saves everything)
                                             │
                                             ▼
                                            END
```

### Language-model calls per run

| DPO decision | Calls before pause | Calls after | Total |
|---|---|---|---|
| RECORD ONLY | 5 | 0 | **5** |
| NOTIFY REGULATOR | 5 | 1 (filing) | **6** |
| NOTIFY REGULATOR AND CUSTOMERS | 5 | 2 (notice + filing) | **7** |

Every call is logged (agent, model, prompt/response size, latency) in the `llm_calls` table. The Audit trail page checks the actual count matches this expected count.

---

## 3. The state (`state.py`)

`IncidentAssessmentState` is a `TypedDict` – the "shared notebook" each node writes into:

| Group | Keys |
|---|---|
| Input | `incident_reference`, `analyst_request`, `analyst_id`, `thread_id` |
| IDs | `run_id` (`RUN-…`), `assessment_id` (`ASM-…`) |
| Facts from tools | `incident_profile`, `campaign_incidents`, `harm_facts`, `attack_facts`, `reporting_facts` |
| Agent outputs | `assessment_plan`, `harm_assessment`, `attack_analysis`, `reporting_advice`, `incident_assessment` |
| After the gate | `dpo_decision`, `customer_notice`, `regulator_filing`, `filing_submitted_at` |
| Log | `llm_call_log: Annotated[list, operator.add]` |

`llm_call_log` uses a **reducer** (`operator.add`): when a node returns `{"llm_call_log": [one_call]}`, LangGraph **appends** it instead of replacing the list. So the log grows by one per agent.

---

## 4. Each node in detail (`nodes.py`)

When `nodes.py` is imported it loads, **once**: the three models, their feature lists, the open incident & notes CSVs, and the Chroma index.

| # | Node | What it does | Writes to DB |
|---|---|---|---|
| 1 | `load_incident_context` | Fetches the incident profile, its notes and its campaign siblings; runs the **harm model** and the **attack model**. Creates `run_id` / `assessment_id`. | `agent_runs` row, status `RUNNING` |
| 2 | `plan_assessment` | Planner agent decides what must be checked, based on the analyst's request. | `llm_calls` |
| 3 | `assess_harm` | Harm analyst explains what drives the harm score, flags if protection state and band disagree. | `llm_calls` |
| 4 | `identify_attack` | Attack analyst judges how far to trust the attack type (if note agreement < ~50 %, it must say so). | `llm_calls` |
| 5 | `check_reporting_duties` | **Tools:** which procedures apply → read their deadlines from the PDFs → compute every clock. **Agent:** reporting advisor explains authorities, deadlines, customer notice. | `llm_calls` |
| 6 | `compose_incident_assessment` | **Tool:** effort model + "does it fit before the earliest deadline?". **Agent:** writer produces the assessment and a recommended disposition. | `security_incidents` row, status `PENDING_DECISION` |
| 7 | `request_dpo_decision` | Marks run `AWAITING_DECISION` and calls **`interrupt()`** – the graph halts. When resumed, the DPO's decision becomes `dpo_decision`. | `agent_runs` updated |
| 8 | `draft_customer_notice` | Reads the attack type's **playbook** for customer steps; notice agent writes it; then **scrubs internal identifiers** out of it. | `llm_calls` |
| 9 | `draft_regulator_filing` | Filing agent writes the regulator filing, including the decision, who took it and why. Filing time = snapshot time. | `llm_calls` |
| 10 | `record_incident_outcome` | Saves the decision, final status, notice/filing text, and computes the **final report due date** (e.g. +1 month after filing for Procedure 02). | `dpo_decisions`, `security_incidents` updated, run `COMPLETE` |

`route_after_decision` simply returns the decision text, and `graph.py` maps it to the next node.

### Identifier scrubbing (node 8)
A customer notice must never contain internal IDs. `_internal_identifiers()` collects the incident ID, analyst IDs, ticket queue and shift codes, campaign reference, linked incident references, and any token with a digit found in the notes (e.g. host names like `BLR-LT-0407`). They're all removed from the notice text after the model writes it – a safety net in case the model ignores the instruction.

---

## 5. The seven agents (`agents.py`)

All use **Google Gemini** (model name and API key from `.env`: `GEMINI_MODEL`, `GEMINI_API_KEY`), temperature `0.2` (consistent, not creative), max 900 output tokens, automatic function calling **disabled**. Transient errors (429/5xx) are retried up to 3 times.

Each agent must reply in a **labelled layout**, which `parse_labelled_response()` reads back into a dict. If a label is missing, it raises an error rather than continuing with half an answer.

| Agent | Role | Reply labels |
|---|---|---|
| `incident_intake_planner` | Decides what must be established. Plans; settles nothing. | `PRIORITY`, `CHECKS`, `RATIONALE` |
| `harm_analyst` | Explains the harm drivers for the DPO. | `DRIVERS`, `PROTECTION`, `HARM SUMMARY` |
| `attack_analyst` | Weighs the attack-type evidence from the notes. | `ATTACK TYPE`, `CONFIDENCE`, `EVIDENCE` |
| `reporting_duty_advisor` | Who must be told and by when – using only the procedure passages. | `AUTHORITIES`, `DEADLINES`, `CUSTOMER NOTICE` |
| `incident_writer` | Writes the assessment the DPO reads + recommends one disposition. | `ASSESSMENT`, `RECOMMENDED DISPOSITION` |
| `customer_notice_agent` | Plain-language notice for customers (after decision). | `SUBJECT`, `NOTICE` |
| `regulator_filing_agent` | The filing for the authorities (after decision). | `SUMMARY`, `SECTIONS`, `CUSTOMERS` |

`read_disposition()` makes sure the writer's recommendation is **exactly** one of `RECORD ONLY`, `NOTIFY REGULATOR`, `NOTIFY REGULATOR AND CUSTOMERS`.

(An eighth name, `response_assistant`, is the chat – see the [RAG README](../knowledge/README.md).)

---

## 6. The tools behind the facts (`tools/`)

| Tool file | Functions | What they decide |
|---|---|---|
| `incident_data_tools.py` | `fetch_incident_profile`, `fetch_incident_notes`, `fetch_campaign_incidents` | Look up one incident, its notes (oldest first), and all incidents in the same campaign. |
| `prediction_tools.py` | `build_harm_facts`, `build_attack_facts`, `estimate_containment_effort` | Model outputs → band, data categories, majority attack type, hours. See [ML README](../modeling/README.md). |
| `retrieval_tools.py` | `read_reporting_duties`, `read_customer_steps` | Deadlines and customer steps from the PDFs. See [RAG README](../knowledge/README.md). |
| `deadline_tools.py` | see below | The reporting rules and clocks. |

### `deadline_tools.py` – the reporting rules

**`select_applicable_procedures`** – which laws/procedures apply:

| Procedure | Applies when… |
|---|---|
| **01 – EU personal data breach** (Lead supervisory authority) | personal data involved **and** EU data subjects > 0 **and** harm band is HIGH or MEDIUM |
| **02 – Significant incident** (National CSIRT) | outage > 60 min **or** customers affected **across the whole campaign** > 100,000 **or** direct loss > €500,000 |
| **03 – India cyber incident** (CERT-In) | hosted in India / EU and India **or** Indian data subjects > 0 |

It also flags `significant_only_with_campaign` – an incident that on its own is small, but becomes "significant" because its campaign siblings together cross 100,000 customers. Easy to miss by hand.

**`compute_procedure_deadlines`** – for every milestone read from the PDF:
- `due_at = clock start + hours` (start is `confirmed_at` for "awareness", `first_alert_at` for "the first alert").
- `hours_remaining` is measured from the queue **snapshot** (`2026-03-14T01:40:00Z`) so results are reproducible.
- Shown in UTC **and** in the hub's local time.
- If the incident isn't confirmed yet, "awareness" clocks are *"Not started"*.
- Picks the **earliest** running milestone.

**`compare_effort_with_deadline`** – `hours_available = hours_remaining − 0.5` (half an hour reserved for writing and filing). If effort needed > hours available → warning *"The estimated effort does not fit…"*; if already overdue → *"The earliest deadline has already passed"*.

**`rank_incident_queue`** – orders the whole queue: incidents with a running clock first (fewest hours left at the top), then the rest by highest harm probability.

**`compute_final_report_due`** – milestones counted "N months after filing" (e.g. Procedure 02's final report).

---

## 7. Pause and resume – how it actually works

1. **Start** (`POST /assessments`): the API creates a unique `thread_id` and calls `workflow.invoke(initial_state, {"configurable": {"thread_id": …}})`. The graph runs nodes 1–7 and stops at `interrupt()`. `SqliteSaver` has saved the full state in `database/checkpoint.db`.
2. **Waiting:** the assessment shows up in the **Decision queue**. The server can even be restarted – nothing is lost.
3. **Resume** (`POST /incidents/{assessment_id}/decision`): the API looks up the stored `thread_id` and calls `workflow.invoke(Command(resume=decision), config)`. LangGraph reloads the state, `interrupt()` returns the decision dict, and the graph continues down the chosen branch to the end.

The decision dict contains `decision`, `decided_by` (display name), `decided_by_role` and `decision_reason` – taken from the signed-in user on the server, not from the browser.

---

## 8. Files

| File | Job |
|---|---|
| `state.py` | The shared state definition. |
| `graph.py` | Nodes, edges, the conditional branch, the SQLite checkpointer. |
| `nodes.py` | The ten steps; loads models/data/index once. |
| `agents.py` | Gemini client, reply parser, the seven prompts. |

⬅ Previous: [RAG](../knowledge/README.md) · ➡ Next: [FastAPI backend](../api/README.md)
