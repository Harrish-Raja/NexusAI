# FastAPI Backend (`api/` + `access/` + `persistence/`)

> **In one sentence:** the backend is a REST API that the dashboard talks to – it signs people in, checks what each role may do, and exposes the queue, incident details, the LangGraph assessment, the DPO decision, the chat and the audit trail, saving everything in a SQLite database.

Run it with:

```bash
python run_database_setup.py   # once: creates tables + demo accounts
python run_api_service.py      # serves http://localhost:8081  (Swagger UI at /docs)
```

---

## 1. Where the backend sits

```
 ┌──────────────────┐   HTTP + X-Session-Token   ┌──────────────────────────────┐
 │ Streamlit        │ ─────────────────────────▶ │ FastAPI  (api/routes.py)     │
 │ dashboard :8501  │ ◀───────────────────────── │  ├─ sign-in guard            │
 └──────────────────┘          JSON              │  ├─ role/area guard          │
                                                 │  ├─ LangGraph workflow       │
                                                 │  ├─ ML models + tools        │
                                                 │  ├─ Chroma (RAG)             │
                                                 │  └─ Gemini                   │
                                                 └──────────────┬───────────────┘
                                                                │ SQLAlchemy
                                                   database/breachwatch.db (SQLite)
                                                   database/checkpoint.db  (LangGraph)
```

The dashboard **never** touches the database, the models or Gemini directly – it only calls this API. All rules are enforced in one place.

---

## 2. What happens at start-up

When `api/routes.py` is imported (i.e. when the server starts):

1. `agentic.nodes` is imported → loads the 3 models, feature lists, open incidents/notes CSVs, and opens the Chroma index.
2. The LangGraph workflow is compiled with its SQLite checkpointer.
3. The three procedures' deadlines are read from the index.
4. The **whole incident queue is ranked once** (harm model + deadlines) and kept in memory, so `GET /queue` is instant.

That's why start-up takes a few seconds, and why all earlier stages (`run_data_pipeline`, `run_model_training`, `run_knowledge_indexing`) must have been run first.

---

## 3. Sign-in (authentication)

### Passwords – `access/password_security.py`
Passwords are never stored. Each account has its own random **salt**, and we store `PBKDF2-HMAC-SHA256(password, salt, 200,000 rounds)`. The many rounds make brute-forcing slow.

### `POST /auth/signin`
1. Look up the username.
2. Hash the offered password with that account's salt.
3. Compare with `hmac.compare_digest` (**constant-time**, so the time taken doesn't leak how many characters matched).
4. If the user doesn't exist, it still hashes against a dummy salt – so "unknown user" and "wrong password" take the same time and give the same message (*"Username or password is not correct"*). An attacker can't discover which usernames exist.
5. On success: create a random **session token** (`secrets.token_urlsafe(24)`), store it on the account, and return it with the user's role, allowed **pages** and **areas**.

### Using the token
Every other request must send the header:

```
X-Session-Token: <token>
```

`POST /auth/signout` clears the token on the server, so it stops working immediately.

### Demo accounts (created by `run_database_setup.py`, from `config.SEED_USER_ACCOUNTS`)

| Role | Usernames | Password |
|---|---|---|
| Data Protection Officer | `ines`, `arjun`, `katarzyna` | `11111111` |
| Security Analyst | `priya`, `tomasz`, `kadri`, `joao`, `meera`, `rasmus` | `22222222` |
| Customer Communications Agent | `ananya`, `mateusz`, `liis`, `beatriz`, `rohan`, `marta` | `33333333` |

---

## 4. Permissions (authorisation) – `access/permissions.py`

Every URL belongs to an **area**, worked out from the path:

| Path starts with | Area |
|---|---|
| `/queue` | intake |
| `/assessments` | assessment |
| `/incidents`, `/audit` | audit |
| `/chat` | chat |
| **POST** ending in `/decision` | **decision** (checked first) |

And each role reaches certain areas:

| Role | intake | assessment | decision | audit | chat |
|---|:-:|:-:|:-:|:-:|:-:|
| Data Protection Officer | ✅ | ✅ | ✅ | ✅ | ✅ |
| Security Analyst | ✅ | ✅ | ❌ | ✅ | ✅ |
| Customer Communications | ❌ | ❌ | ❌ | ❌ | ✅ |

`authorize_request` is a FastAPI **dependency** attached to the whole `guarded_router`, so it runs before every protected endpoint:
- no/invalid token → **401** *"Sign in first"*
- role can't use the area → **403** e.g. *"A Security Analyst may not use the decision area"*

So an analyst **cannot** record a DPO decision even by calling the API directly – the button being hidden in the UI is not the only protection.

---

## 5. Endpoints

| Method & path | Area | What it does | Key inputs | Returns |
|---|---|---|---|---|
| `POST /auth/signin` | public | Sign in | `{username, password}` | token, role, pages, areas |
| `POST /auth/signout` | public (needs token) | Sign out | header | `{signed_out: true}` |
| `GET /queue` | intake | The ranked queue | `?harm_band=ALL\|HIGH\|MEDIUM\|LOW` (required) | list of rows: reference, hub, harm prob/band, procedures, earliest milestone… |
| `GET /queue/{incident_reference}` | intake | Everything the tools know about one incident | – | profile, notes, campaign, harm, attack, applicability, duties, deadlines, effort |
| `POST /assessments` | assessment | **Runs the LangGraph workflow until the DPO pause** | `{incident_reference, analyst_request (≥5 chars)}` | **201** + the saved assessment (status `PENDING_DECISION`) |
| `GET /incidents` | audit | Assessments filtered by status | `?status_value=PENDING_DECISION&status_value=RECORDED…` (required, repeatable) | list of assessments |
| `POST /incidents/{assessment_id}/decision` | **decision** | DPO decision → **resumes the workflow** | `{decision, decision_reason (≥10 chars)}` | the updated assessment (with notice/filing if any) |
| `GET /incidents/{assessment_id}/calls` | audit | LLM call budget for the run | – | calls made vs expected, latency, each call |
| `POST /chat/ask` | chat | Ask the response assistant (RAG) | `{question (≥3 chars)}` | type, answer, sources, refused? |
| `GET /chat/turns` | chat | **My own** chat history | `?turn_limit=1..200` | turns |
| `GET /audit/chat-turns` | audit | **Everyone's** recent chat turns | `?turn_limit=1..200` | turns |

### Error codes you may see
| Code | Meaning |
|---|---|
| 401 | Not signed in / bad token / wrong password |
| 403 | Your role can't use this area |
| 404 | Incident not in the queue / no such assessment |
| 409 | Assessment already decided (a decision can't be made twice) |
| 422 | Request body/query failed validation (e.g. reason too short, wrong band name) |
| 502 | Something downstream failed – usually Gemini (missing key, quota, bad reply). The message says what. |

### The two workflow endpoints in detail

**`POST /assessments`**
1. Check the incident is in the queue (else 404).
2. New `thread_id = "<reference>-<random>"`.
3. `assessment_workflow.invoke({...}, {"configurable": {"thread_id": …}})` – runs nodes 1–7 (5 Gemini calls). This takes a while (the dashboard allows 180 s).
4. Confirm the graph is paused at `request_dpo_decision`; otherwise 502.
5. Return the assessment record.

**`POST /incidents/{assessment_id}/decision`**
1. 404 if unknown; 409 if already decided.
2. Build the decision from the **signed-in user** (name and role come from the server, not the request).
3. `assessment_workflow.invoke(Command(resume=decision), config)` – continues the paused graph down the chosen branch.
4. Return the final record: status, customer notice, regulator filing, final report due date.

---

## 6. Request/response shapes – `api/schemas.py`

Pydantic models validate every request and shape every response. Some useful rules they enforce automatically:

- `harm_band` must be one of `HIGH`, `MEDIUM`, `LOW`, `ALL`.
- `decision` must be exactly one of `RECORD ONLY`, `NOTIFY REGULATOR`, `NOTIFY REGULATOR AND CUSTOMERS`.
- `analyst_request` ≥ 5 chars, `decision_reason` ≥ 10 chars, `question` ≥ 3 chars.

FastAPI uses these to generate the **Swagger UI** at `http://localhost:8081/docs`.

### Trying it in Swagger
1. Open `/docs` → `POST /auth/signin` → *Try it out* → `{"username": "ines", "password": "11111111"}` → copy `session_token`.
2. For any other endpoint, paste the token in the **X-Session-Token** header field.
3. e.g. `GET /queue` with `harm_band=HIGH`.

---

## 7. The database – `persistence/`

SQLite file `database/breachwatch.db`, accessed with **SQLAlchemy 2.0** ORM.

| File | Job |
|---|---|
| `database.py` | Creates the engine and `SessionFactory` (no tables created on import). |
| `models.py` | The six table definitions. |
| `repository.py` | **Every** read/write. Returns plain dicts, so the rest of the code never handles ORM objects. |
| `../run_database_setup.py` | Creates missing tables and seeds accounts. Safe to run again – it never overwrites. |

### The six tables

| Table | One row per… | Main columns | Written by |
|---|---|---|---|
| `user_accounts` | person | username, display_name, role, password_hash, password_salt, session_token | setup script, sign-in/out |
| `agent_runs` | workflow run | run_id, thread_id, incident_reference, analyst_id, analyst_request, status (`RUNNING` → `AWAITING_DECISION` → `COMPLETE`), llm_call_count | nodes 1, 7, 10 |
| `security_incidents` | assessment | assessment_id, harm prob/band, attack type & agreement, procedures, earliest deadline, hours remaining, effort, can file in time, customer notice required, plan text, assessment text, recommendation, **status**, notice text, filing text, filed at, final report due | node 6 (create), node 10 (update) |
| `dpo_decisions` | decision | assessment_id, decision, decided_by, decided_by_role, decision_reason, decided_at | node 10 |
| `llm_calls` | Gemini call | run_id, agent_name, model_name, prompt/response characters, latency ms | every agent + chat |
| `chat_turns` | chat question | username, role, question, type, incident ref, answer, sources, closest distance, was_refused | `/chat/ask` |

`fetch_incident_records` joins `security_incidents` + `agent_runs` + `dpo_decisions` so one record has the assessment, its thread and its decision together.

### Incident status lifecycle

```
PENDING_DECISION ──RECORD ONLY─────────────────────▶ RECORDED
        │
        ├────────NOTIFY REGULATOR──────────────────▶ NOTIFIED
        │
        └────────NOTIFY REGULATOR AND CUSTOMERS────▶ NOTIFIED_AND_DISCLOSED
```

Only `NOTIFIED_AND_DISCLOSED` incidents can be discussed (via the published notice) with Customer Communications staff in the chat.

---

## 8. Files

| File | Job |
|---|---|
| `api/routes.py` | The FastAPI app, guards and all endpoints. |
| `api/schemas.py` | Pydantic request/response models. |
| `access/password_security.py` | PBKDF2 password hashing. |
| `access/permissions.py` | Path → area → allowed? |
| `persistence/*` | Database engine, tables, repository. |
| `run_api_service.py` | Starts uvicorn on `0.0.0.0:8081`. |

⬅ Previous: [LangGraph](../agentic/README.md) · ➡ Next: [Streamlit dashboard](../dashboard/README.md)
