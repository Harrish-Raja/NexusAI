# RAG – Retrieval-Augmented Generation (`knowledge/` + `chat/` + `tools/retrieval_tools.py`)

> **In one sentence:** we turn the company's 18 policy PDFs into a searchable "meaning index", so the system can look up the *exact* passage that answers a question – and then either read a rule straight out of it, or hand it to the language model to write an answer grounded in it.

Build the index with (after the data pipeline and model training):

```bash
python run_knowledge_indexing.py
```

---

## 1. What is RAG and why use it?

A language model (Gemini here) knows a lot about the world in general, but nothing about **Merrowfield Digital's own** procedures – and if you ask it anyway it may confidently invent a deadline. That is unacceptable for legal reporting.

**RAG** fixes this in two steps:

1. **Retrieve** – search our own documents for the passages most relevant to the question.
2. **Generate** – give the model *only those passages* and tell it to answer from them and nothing else.

So the model becomes a reader of our documents, not an inventor of facts.

This project goes one step further: for the most critical facts (reporting **deadlines**), it doesn't let the model read them at all – the code extracts them from the retrieved passage with exact sentence patterns. More on that in section 5.

---

## 2. The documents (`data/docs/`)

| Kind | Documents | Used for |
|---|---|---|
| **Procedures** (3) | `Procedure-01-EU-Personal-Data-Breach.pdf`, `Procedure-02-Significant-Incident-Reporting.pdf`, `Procedure-03-India-Cyber-Incident-Reporting.pdf` | Reporting deadlines, customer-notice exemption |
| **Playbooks** (8, one per attack type) | e.g. `Playbook-Ransomware.pdf`, `Playbook-Phishing-And-Credential-Compromise.pdf` | What customers should do; response steps |
| **Policies & standards** | Incident Response Plan, Customer Communication Policy, Data Classification Standard, Sensitive Data Handling Standard | Chat answers |
| **Casebook** | `Past-Incidents-Casebook.pdf` | Indexed, but **excluded from chat** (it describes real past incidents) |

---

## 3. Building the index – step by step

```
 18 PDFs
   │  pdf_extraction.py      pypdf reads every page → one long text per document
   ▼
 raw text
   │  chunk_splitting.py     1) remove the repeating page header ("Document … | Rev 3 … Review cycle: …")
   │                         2) cut at numbered headings 1. 2. 3. … → sections
   │                         3) cut each section into passages of whole sentences
   ▼
 passages (chunks)           ~700 characters each, ~120 characters overlap
   │  vector_index.py        embed each passage with all-MiniLM-L6-v2
   ▼
 ChromaDB                    database/chroma_store/  (collection "incident_response_documents")
```

### 3.1 Splitting into sections
Headings are found with a pattern like `"3. The first hour"`. To avoid being fooled by a list item `"2. Call the DPO"` in the middle of a section, a heading only counts if the numbers run **in sequence** – 1, then 2, then 3. So each section is a real section of the document.

### 3.2 Splitting into chunks (passages)
A whole section can be too long to be a good search result, so each section is cut into **passages of whole sentences** of up to **700 characters**. Neighbouring passages **overlap by ~120 characters** (the last sentence or two are repeated at the start of the next passage), so a fact sitting on a boundary is never cut in half.

Each passage gets a readable ID: `Playbook-Ransomware-s003-c002` = *document → section 3 → chunk 2*.

### 3.3 Embedding – turning meaning into numbers
An **embedding model** (`sentence-transformers/all-MiniLM-L6-v2`, runs on CPU) turns a piece of text into a list of 384 numbers. Texts with **similar meaning** get **similar numbers**, even if they use different words – "deadline to tell the regulator" lands near "notification to the supervisory authority is due within 72 hours".

One clever detail: all eight playbooks share the same headings (`"3. The first hour"`), so a passage on its own doesn't say *which* playbook it is from. When embedding, the passage is prefixed with the document title – `"Playbook Ransomware: …"` – so the meaning includes the attack type. The **stored** text stays exactly as in the PDF.

### 3.4 The vector store – ChromaDB
Chroma stores each passage with its embedding and its `document_name`. It uses **cosine distance**: `0` = same meaning, larger = less related. It can also **filter by document**, which is used heavily below.

---

## 4. Searching (`vector_index.search_documents`)

```python
search_documents(collection, ["what customers should do"], 3, {"$eq": "Playbook-Ransomware.pdf"})
```

- The question is embedded with the same model.
- Chroma returns the nearest passages, optionally limited to one document (`$eq`) or excluding some (`$nin`).
- Each result has: `chunk_id`, `chunk_text`, `document_name`, `distance`.

---

## 5. Use #1 – Reading reporting deadlines (no language model!)

`procedure_reading.py` – this is "RAG without the G".

Deadlines decide legal obligations, so we never let a model paraphrase them. Instead:

1. Search **only inside that procedure's own PDF** (document filter) for *"which reports are due, within how many hours, counted from which moment"*.
2. Run exact sentence patterns over the 3 returned passages.

The procedures are written with fixed sentences, and these are what the patterns read:

| Procedure | Sentence in the PDF | What the code extracts |
|---|---|---|
| 01 – EU personal data breach | *"The notification to the lead supervisory authority is due within 72 hours of awareness."* | 72 h from `confirmed_at` |
| 02 – Significant incident | *"The early warning is due within 24 hours of awareness."* | 24 h from `confirmed_at` |
| | *"The incident notification is due within 72 hours of awareness."* | 72 h from `confirmed_at` |
| | *"The final report is due one month after the incident notification is filed."* | +1 month after filing |
| 03 – India cyber incident | *"The incident report to CERT-In is due within 6 hours of the first alert."* | 6 h from `first_alert_at` |
| 01 – exemption | *"Customers need not be told where the data protection state is Encrypted."* | Encrypted data → no customer notice |

The phrase *"awareness"* maps to the `confirmed_at` column and *"the first alert"* maps to `first_alert_at` (see `CLOCK_START_COLUMN_BY_PHRASE` in config).

If a procedure's sentences can't be read, the code **raises an error** rather than guessing. `run_knowledge_indexing.py` prints every deadline it read and checks each came from **its own** document – a quick visual proof the index is right.

**Why this is good:** change "72 hours" to "48 hours" in the PDF, rebuild the index, and every deadline in the system follows – no code change.

---

## 6. Use #2 – Customer steps for the notice

`tools/retrieval_tools.read_customer_steps(attack_type)` searches **only the playbook for that attack type** for *"what should affected customers do now"* and returns the top 3 passages. The customer-notice agent writes the notice from these, so the advice ("reset your password", "watch for suspicious emails") comes from our playbook, not from the model's imagination.

`read_reporting_duties()` combines use #1 for all applicable procedures, reads the exemption, and decides `customer_notice_required` (Procedure 01 applies **and** the harm band is HIGH **and** the data is not in an exempt protection state).

---

## 7. Use #3 – The Response chat (full RAG with Gemini)

This powers the **Response chat** page. Files: `chat/response_assistant.py`, `chat/conversation_memory.py`, `chat/incident_status_answer.py`.

```
 question ─┐
           ├─▶ conversation_memory: last 3 Q&As of THIS user (from the database)
           │
           ├─▶ search #1: the question itself                     ┐ 8 passages each,
           ├─▶ search #2: earlier questions + this question        ┘ casebook excluded
           │        → merge, keep each passage once, nearest first
           │
           ├─▶ ONE Gemini call with: rules + history + numbered passages + question
           │        model must reply in a fixed layout:
           │          QUESTION TYPE: PROCEDURE | REGULATION | CUSTOMER ADVICE |
           │                         INCIDENT STATUS | OUT OF SCOPE
           │          INCIDENT REFERENCE: INC-… or NONE
           │          ANSWER: …
           │
           └─▶ guardrails decide what the user actually sees
```

### Why two searches?
For a follow-up like *"and what about India?"*, the question alone is too vague. Search #2 adds the user's previous questions, so the retrieval understands the context.

### The guardrails (the important part)
The model's answer is shown **only if all** of these hold:
1. The type is PROCEDURE, REGULATION or CUSTOMER ADVICE.
2. The model didn't decline.
3. The closest passage has distance **≤ 0.6** – i.e. our documents really contain something relevant.

Otherwise the user gets a fixed refusal: *"I can only answer from Merrowfield's reporting procedures, policies, standards and playbooks…"*. Every answer lists the **source documents** it came from.

### Questions about a specific incident
If the question names an incident (pattern `INC-BLR-1234567`), the type is forced to **INCIDENT STATUS** and the model's answer is thrown away. `incident_status_answer.py` answers from the **database** instead, with **no model call**, depending on who's asking:

| Role | What they get |
|---|---|
| Security Analyst, DPO | A summary of the latest assessment: harm, attack, procedures, deadline, effort, recommendation, decision, filing. |
| Customer Communications | **Only** the published customer notice – and only if the incident status is `NOTIFIED_AND_DISCLOSED`. Otherwise: *"I can't share anything about that incident…"* |

This stops confidential incident details leaking to people who shouldn't see them, even if they phrase the question cleverly.

### Memory
`conversation_memory.recall_conversation` reads the user's last **3** turns from the `chat_turns` table – the history comes from the server's record, never from the browser, so it can't be tampered with. Every turn (question, type, answer, sources, distance, refused?) is saved and visible in the **Audit trail**.

---

## 8. Files

| File | Job |
|---|---|
| `knowledge/pdf_extraction.py` | Reads text out of every PDF (alphabetical order). |
| `knowledge/chunk_splitting.py` | Sections → overlapping sentence passages with IDs. |
| `knowledge/vector_index.py` | The **only** file that talks to ChromaDB: build, open, search. |
| `knowledge/procedure_reading.py` | Extracts deadlines/exemption from a procedure's own passages. |
| `tools/retrieval_tools.py` | Reporting duties + playbook customer steps for the agents. |
| `chat/response_assistant.py` | The chat's retrieve → one model call → guardrails. |
| `chat/conversation_memory.py` | Last 3 exchanges of the user, from the database. |
| `chat/incident_status_answer.py` | Role-aware incident answers from the record. |
| `../run_knowledge_indexing.py` | Builds the index and proves each procedure reads correctly. |

Key settings (all in `config/config.py`): `CHUNK_SIZE_CHARACTERS=700`, `CHUNK_OVERLAP_CHARACTERS=120`, `RETRIEVAL_RESULT_COUNT=8`, `CHAT_MAX_RELEVANCE_DISTANCE=0.6`, `EMBEDDING_MODEL_NAME="all-MiniLM-L6-v2"`.

⬅ Previous: [Machine learning](../modeling/README.md) · ➡ Next: [LangGraph agents](../agentic/README.md)
