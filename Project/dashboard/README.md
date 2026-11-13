# Streamlit Dashboard (`dashboard/` + `run_dashboard.py`)

> **In one sentence:** the dashboard is the screen people actually use – sign in, see the incident queue ordered by urgency, inspect an incident, ask the AI agents for an assessment, let the DPO decide, review the audit trail, and chat with the policy assistant.

Run it with (the API must already be running):

```bash
python run_api_service.py                  # terminal 1
python -m streamlit run run_dashboard.py   # terminal 2 → http://localhost:8501
```

The dashboard holds **no business logic**. Every figure comes from the API via `dashboard/api_client.py`. If the API is down you'll see: *"The BreachWatch AI service at http://localhost:8081 cannot be reached… Start it with: python run_api_service.py"*.

---

## 1. Signing in and what each role sees

Sign in with a demo account (full list in section 3 of the [API README](../api/README.md)), e.g. `ines` / `11111111` (DPO) or `priya` / `22222222` (Analyst).

The pages in the top navigation depend on your role:

| Page | Security Analyst | Data Protection Officer | Customer Communications |
|---|:-:|:-:|:-:|
| Incident queue | ✅ | ✅ | ❌ |
| Incident detail | ✅ | ✅ | ❌ |
| Decision queue | ❌ | ✅ | ❌ |
| Audit trail | ✅ | ✅ | ❌ |
| Response chat | ✅ | ✅ | ✅ |

Your name and role show at the top right with a **Sign out** button. If your session expires (401), you're sent back to the sign-in form.

---

## 2. Incident queue

**Purpose:** "What should the desk work on first?"

### What you see
**Five metric cards:**
| Card | Meaning |
|---|---|
| Open incidents | How many incidents match the current filter. |
| Deadline already passed | Incidents whose earliest reporting deadline is already behind the snapshot time. |
| Next deadline | Hours until the soonest deadline that hasn't passed yet. |
| Running before confirmation | Clocks already ticking although the incident isn't confirmed (e.g. India's 6 h from the *first alert*). |
| Significant through a campaign | Incidents that are only "significant" because their campaign together crosses 100,000 customers. |

**The table** – sorted with the **fewest hours left at the top**; incidents with no running clock come last, highest harm first. Columns: Incident, Hub, Band, Harm probability (progress bar), Hours left, Earliest duty (procedure – milestone – authority), Due (hub time), Procedures, Overdue, Awaiting confirmation.

### Actions
1. **Harm band filter** – `ALL` / `HIGH` / `MEDIUM` / `LOW`.
2. **Clocks filter** – Every incident / Still running / Deadline passed / No clock running.
3. **Click a row** → it's remembered and will open on the **Incident detail** page. Then click *Incident detail* in the navigation.

> All "hours left" are measured from the snapshot time `2026-03-14T01:40:00Z`, so everyone sees the same numbers.

---

## 3. Incident detail

**Purpose:** "Everything we know about this one incident – and ask the agents to assess it."

At the top: an **Incident** drop-down (pre-selected from the queue), then badges (harm band & probability, hub, confirmation status, campaign, containment status) and a one-line summary (system, region, attack vector, detection source, severity).

The figures here come from the **same tools the agents use**, so what you see is exactly what the assessment will be given.

### Tab 1 – Reporting clock
One card per reporting milestone:
- Procedure and milestone, the authority to tell (e.g. *Procedure 03 – India cyber incident – incident report to CERT-In*).
- **Earliest** / **Overdue** badges.
- "Within **6 hours** of the first alert", when it started.
- Due time in the **hub's local time** and in UTC, and **hours left** (red if overdue).
- *"Read from Procedure-03-…pdf"* – proof the rule came from the document.
- If the incident isn't confirmed: *"Not started – the incident is awaiting confirmation"*.

**Expander "Why each procedure applies"** – the reasons, e.g. *"a service outage of 95 minutes, more than 60"*; a warning if it's significant only through its campaign; whether customers must be told and which protection would excuse it.

*Action:* read only.

### Tab 2 – Incident
Four boxes of raw facts: **Timeline** (first alert, triage started, confirmed), **Systems** (class, region, count, internet facing, supplier, backups), **People and data** (EU/Indian data subjects, records, customers, protection, MFA, exfiltration), **Impact** (outage, direct loss, attacker still active, campaign).

*Action:* read only.

### Tab 3 – Harm and attack
- **Left:** the harm probability (big number), its band, and the data categories involved / high-impact / sensitive.
- **Right:** the attack type the notes name and the agreement (*"Named by 75 % of 4 notes"*).
- **Below:** every analyst note, each labelled with the attack type the model gave *that* note – so you can see where the notes disagree.

*Action:* read only.

### Tab 4 – Effort
Four metrics:
| Metric | Meaning |
|---|---|
| Estimated by the model | Raw effort model output (hours). |
| Effort needed | After the ×1.5 sensitive-data multiplier, if it applies. |
| Hours available | Hours to the earliest deadline minus 0.5 h kept for writing and filing. |
| Complete report in time | yes / no / – (no running deadline). |

A warning appears if the work doesn't fit or the deadline has passed. The caption shows the planned start time in hub time, whether it's out of hours, and which values were **assumed rather than known**.

*Action:* read only.

### Tab 5 – Assess ⭐ (main action)
1. Type **what you want looked at** (≥ 5 characters), e.g. *"Do we need to notify anyone and can we make the deadline?"*
2. Click **Start the assessment**.
3. Wait – five agents run (5 Gemini calls). Nothing is filed and no customer is told.
4. You get: *"Assessment ASM-… is waiting on the Decision queue. The agents recommend NOTIFY REGULATOR…"* and an expander with what the agents wrote.

Behind the scenes this calls `POST /assessments`, which runs the LangGraph workflow until it pauses for the DPO.

---

## 4. Decision queue (DPO only)

**Purpose:** "A human decides what happens."

### What you see
- A table of assessments **waiting for a decision**: Assessment ID, Incident, Hub, Band, Recommended, Earliest deadline, Hours left, Opened.
- **Left panel** for the selected row (first row by default):
  - Badges: harm band & probability, attack type, **Recommended:** disposition.
  - The **assessment text** the agents wrote.
  - A box with procedures, earliest deadline and hours left, effort needed and "complete report in time", and whether customers must be told under Procedure 01.
  - Expander: what the analyst asked and the planner's plan.
- **Right panel – "Your decision"** form.

### Actions
1. **Select a row** in the table to choose an assessment.
2. **Read** the assessment and facts.
3. Pick a **disposition**:
   - `RECORD ONLY` – log it, tell nobody (0 more model calls).
   - `NOTIFY REGULATOR` – draft the regulator filing (1 more call).
   - `NOTIFY REGULATOR AND CUSTOMERS` – draft the customer notice **and** the filing (2 more calls).
4. Write **your reason, in your own words** (≥ 10 characters). It's stored and quoted in the filing.
5. Click **Record the decision**.

### Result
*"NOTIFY REGULATOR AND CUSTOMERS recorded. The incident now stands at NOTIFIED_AND_DISCLOSED."* plus expanders with the **Customer notice** and **Regulator filing**, and *"Final report due …"* when a procedure requires one.

You **can** choose differently from the recommendation – the AI only recommends. Each assessment can be decided only once.

Behind the scenes: `POST /incidents/{id}/decision` resumes the paused LangGraph workflow.

---

## 5. Audit trail

**Purpose:** "Show me what was decided, by whom, why, and what the AI did."

### Tab "Decided incidents"
- Metrics: total decided, and how many of each disposition.
- Table: Incident, Hub, Band, Recommended, Decided, By, Filed, Final report due, Status.
- **Click a row** to see:
  - The decision badge, who decided, their role, when, and their **reason**.
  - ⚠️ A warning if the officer **overrode** the agents' recommendation.
  - Expanders with the **Customer notice** and **Regulator filing** as sent.
  - **Model calls** – *"6 made, 6 expected for this path (as expected); 14,230 ms in all"* and a table of every call (agent, model, sizes, latency). This proves the AI did exactly the work it was supposed to, no more.

### Tab "Response assistant log"
The 25 most recent chat questions from **everyone**: when, who (and role), kind, incident, refused?, question, answer, source documents. Lets a supervisor check nobody was given information they shouldn't have.

*Actions:* select rows, open expanders – read only.

---

## 6. Response chat

**Purpose:** "Ask a question about our procedures, rules or customer advice and get an answer **only** from our documents."

### What you see
- Your previous conversation (last 20 turns), each answer with a coloured **type badge** – PROCEDURE, REGULATION, CUSTOMER ADVICE, INCIDENT STATUS or OUT OF SCOPE – a **Refused** badge if it was refused, and *"From <documents>"*.
- A side panel explaining what the assistant answers, and that it remembers your last 3 exchanges. Customer Communications users are told incidents are only discussed via the published notice.

### Actions
Type in the chat box at the bottom (≥ 3 characters). Example questions:

| Try asking | Expected type |
|---|---|
| *"What is the deadline to notify the lead supervisory authority?"* | REGULATION / PROCEDURE |
| *"What should customers do after a phishing incident?"* | CUSTOMER ADVICE |
| *"And for ransomware?"* (follow-up – uses memory) | CUSTOMER ADVICE |
| *"What's the status of INC-BLR-0041874?"* | INCIDENT STATUS – answer depends on your role |
| *"Who won the football last night?"* | OUT OF SCOPE – refused |

Each question costs one Gemini call (incident status answers come from the database).

---

## 7. A complete walkthrough (5 minutes)

1. Sign in as **`priya` / `22222222`** (Security Analyst).
2. **Incident queue** → filter `HIGH` → click the top row.
3. **Incident detail** → look at *Reporting clock*, *Harm and attack*, *Effort*.
4. **Assess** tab → type a request → **Start the assessment** → wait for "waiting on the Decision queue".
5. **Sign out**, sign in as **`ines` / `11111111`** (DPO).
6. **Decision queue** → select the assessment → pick `NOTIFY REGULATOR AND CUSTOMERS` → write a reason → **Record the decision** → read the notice and filing.
7. **Audit trail** → *Decided incidents* → click the row → check the reason and "7 made, 7 expected".
8. Sign in as **`ananya` / `33333333`** (Communications) → **Response chat** → ask about that incident reference: you get the published notice only. Ask about an undecided one: refused.

---

## 8. Files

| File | Page / job |
|---|---|
| `../run_dashboard.py` | Sign-in form, role-based navigation, sign-out, error handling. |
| `api_client.py` | `call_service()` – every HTTP call; adds the session token; turns errors into `ServiceError` / `ServiceUnreachable`. |
| `incident_queue.py` | Incident queue page (+ shared `badge()` helper). |
| `incident_detail.py` | Incident detail page with its 5 tabs. |
| `decision_queue.py` | Decision queue page and decision form. |
| `audit_trail.py` | Audit trail page with its 2 tabs. |
| `response_chat.py` | Response chat page. |

⬅ Previous: [FastAPI backend](../api/README.md)
