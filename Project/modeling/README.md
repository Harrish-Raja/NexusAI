# Machine Learning (`modeling/` + `tools/prediction_tools.py`)

> **In one sentence:** we train three models – one says *how likely people are to be harmed*, one says *how many hours containment will take*, and one reads analyst notes and says *what kind of attack it is* – and then turn their raw outputs into facts the business can act on.

Run training with (after the data pipeline):

```bash
python run_model_training.py
```

---

## 1. The three models at a glance

| | Harm model | Effort model | Attack model |
|---|---|---|---|
| **Question it answers** | Are the people whose data is involved at **high risk**? | How many **analyst hours** will containment take? | Which of **8 attack types** does this note describe? |
| **Type of problem** | Binary classification (yes/no) | Regression (a number) | Text classification (NLP, 8 classes) |
| **Learns from** | `security_incidents_*_clean.csv` | `containment_effort_*_clean.csv` | `analyst_incident_notes_clean.csv` |
| **Target column** | `high_risk_to_individuals` (1/0) | `containment_hours` | `attack_type` |
| **Algorithm** | HistGradientBoostingClassifier | HistGradientBoostingRegressor on **log** hours | TF-IDF (words + characters) → LinearSVC |
| **Main metric** | ROC AUC | R² | Macro F1 |
| **Minimum required** | 0.75 | 0.55 | 0.75 |
| **Achieved (eval)** | **0.885** | **0.731** | **0.881** |

The minimums live in `config/config.py`. The training script **reports** a model that misses its minimum – it never hides it.

---

## 2. Files

| File | Job |
|---|---|
| `harm_model.py` | Builds and trains the harm classifier. |
| `effort_model.py` | Builds and trains the effort regressor. |
| `attack_model.py` | Builds and trains the text classifier. |
| `model_evaluation.py` | Scores each model on **both** training and evaluation data, and computes the gap between them. |
| `model_registry.py` | Saves/loads models (`joblib`), feature lists and metrics. |
| `../run_model_training.py` | Runs all three, saves everything, prints a summary. |
| `../tools/prediction_tools.py` | Uses the trained models on **live** incidents and turns outputs into business facts. |

What gets written:

```
output/models/harm_model.pkl        ← whole pipeline (preparation + model)
output/models/effort_model.pkl
output/models/attack_model.pkl
output/models/harm_features.json    ← the exact list & order of input columns
output/models/effort_features.json
output/model_metrics.json           ← every score, for every model
```

---

## 3. Harm model – "is this incident dangerous for people?"

### Concept
Each past incident is labelled `high_risk_to_individuals = 1` or `0`. The model learns which combinations of facts tend to mean high risk – for example *plaintext* health data that was *confirmed exfiltrated* is far worse than *encrypted* contact details that never left the building.

### Algorithm: Histogram Gradient Boosting
Gradient boosting builds hundreds of small decision trees **one after another**. Each new tree focuses on the mistakes the previous trees made. "Histogram" means numbers are grouped into bins first, which makes it very fast on tens of thousands of rows.

Settings: `max_iter=300` trees, `learning_rate=0.06`, `l2_regularization=1.0` (stops it from over-fitting), and **native categories** (it splits on categories directly instead of pretending they are numbers).

### Inputs
Every column except the identifiers and the target – 37 features such as `affected_system_class`, `data_protection_state`, `exfiltration_confirmed_flag`, `records_affected_count`, `health_data_flag`, `mfa_enforced_flag`, …

### Metric: ROC AUC = 0.885
ROC AUC answers: *"If I pick one truly high-risk incident and one truly low-risk incident at random, how often does the model give the high-risk one the higher score?"* 0.5 = coin flip, 1.0 = perfect. 0.885 is strong.

Also reported: accuracy, precision, recall, F1, and the **train-vs-eval gap** (0.026 – small, so it is not memorising).

### From probability to a band
`prediction_tools.harm_band_for()` turns the probability into a band people can act on:

| Probability | Band |
|---|---|
| ≥ 0.60 | **HIGH** |
| ≥ 0.30 | **MEDIUM** |
| < 0.30 | **LOW** |

`build_harm_facts()` also lists which *kinds* of data are involved (from the `*_flag` columns): all data categories, the **high-impact** ones (credentials, government ID, payment tokens, health, children's accounts) and the **sensitive** ones (government ID, health, children's accounts).

---

## 4. Effort model – "how long will containment take?"

### Concept
Past containment records say how many hours the work took, plus facts like how many systems and endpoints were affected, whether forensic images were needed, the analyst's grade and workload, etc. The model learns to predict the hours.

### Why predict *log* hours?
Containment times are **skewed**: most take a few hours, a few take 100+. Predicting `log(hours)` and converting back with `exp()` (`TransformedTargetRegressor`) means:
- big outliers don't dominate the training,
- the prediction can **never be negative** (exp of anything is positive).

### Metric: R² = 0.731, MAE = 5.0 hours
- **R²** = how much of the variation in hours the model explains (1.0 = all of it). 0.73 is good for human-effort data.
- **MAE** (mean absolute error) = on average the estimate is ~5 hours off. The **baseline** (always guessing the average) is ~11.7 hours off, so the model more than halves the error.

### Predicting for a *live* incident (the tricky bit)
An open incident doesn't yet have all the columns the effort model was trained on (nobody knows yet how many forensic images will be taken). `estimate_containment_effort()` fills the input row in this priority order:

1. **Facts we can derive now** – attack type (from the attack model), harm band (from the harm model), number of data categories, number of regions, the weekday and whether it starts **out of hours**.
   Containment is assumed to start at the queue snapshot time (`2026-03-14T01:40:00Z`) converted to the **hub's own time zone** – 01:40 UTC is 07:10 in Bengaluru, 03:40 in Tallinn, 02:40 in Warsaw and 01:40 in Lisbon. Working hours are 08:00–18:00 on weekdays. The snapshot falls on a Saturday, so every hub's start counts as out of hours.
2. **Planning assumptions** from config – e.g. 1 endpoint, 5 log sources, a Senior Analyst with 4 open incidents, no forensic image.
3. **Baselines** – `sensitive_data_review_flag = No` (see the next point).
4. **Anything else** is taken from the incident's own record.

Then a business rule: if **sensitive data** is involved, the estimate is multiplied by **1.5**. The model is deliberately asked for the "no sensitive review" number and the multiplier is applied on top, so the extra effort is explicit and visible on screen rather than buried in the model.

The output says which values were **assumed rather than known**, so nobody mistakes an assumption for a fact.

---

## 5. Attack model – "what kind of attack is this?" (NLP)

### Concept
Analysts write free-text notes like:

> *"A seller support agent clicked a link in an invoice email and entered credentials on a lookalike login page (merrowfield-login.net)."*

The model reads the note and outputs one of: Phishing and credential compromise, Ransomware, Other malware, Misdirected disclosure, Lost or stolen device, Insider misuse, Cloud misconfiguration, Denial of service.

### How text becomes numbers: TF-IDF
A model can't read words, so each note becomes a long vector of numbers. **TF-IDF** gives a word a high weight if it appears in *this* note but is rare across *all* notes (so "lookalike" matters, "the" doesn't). Two views are combined (`FeatureUnion`):

- **Word n-grams (1–2 words)** – catches phrases like `"lookalike domain"`, `"encrypted files"`.
- **Character n-grams (3–5 letters)** – catches spelling variants, typos and fragments like `"ransom"` inside `"ransomware"`, or host names like `BLR-LT-0407`.

### Classifier: Linear SVM (`LinearSVC`)
Finds the boundaries that best separate the 8 classes in that huge vector space. It is fast and works very well for text. `class_weight="balanced"` stops it from favouring the most common attack type.

### Train/eval split
There's only one notes file, so it is split **80% train / 20% eval**, **stratified** (each attack type keeps the same share in both halves), with a fixed random seed (42) so results are reproducible.

### Metric: Macro F1 = 0.881
F1 balances precision ("when it says Ransomware, is it right?") and recall ("does it find all the Ransomware?"). **Macro** = average over the 8 classes equally, so a rare class counts as much as a common one. Every class is above 0.84.

### From many notes to one answer
An incident usually has several notes. `build_attack_facts()`:
1. Classifies **each** note separately.
2. Picks the type **most notes agree on** (a tie goes to the type seen first).
3. Reports the **agreement** – e.g. 3 of 4 notes → `0.75`. Low agreement is a warning that the notes don't settle the attack type, and the attack-analyst agent is told to say so.

---

## 6. Why save the feature list separately?

`harm_features.json` and `effort_features.json` store the exact columns (and order) the model was trained on. When scoring a live incident, the row is **re-indexed** to that list: missing columns become gaps (which the pipeline's imputer fills), extra columns are ignored. This makes prediction robust even if the live CSV has a column more or less.

## 7. Overfitting check

For every model, `model_evaluation.py` scores the **training** data too, and reports `generalisation_gap = train score − eval score`. Small gaps (0.03, 0.05, 0.03 here) mean the models learned general patterns rather than memorising the training rows.

---

## 8. Where the models are used later

```
                   ┌──────────────┐
open incident ───▶ │  Harm model  │──▶ probability + band ──┐
                   └──────────────┘                          │
                   ┌──────────────┐                          ├─▶ which procedures apply,
analyst notes ───▶ │ Attack model │──▶ attack type + agreement│   customer notice needed?
                   └──────────────┘                          │   (tools/deadline_tools.py)
                   ┌──────────────┐                          │
both above ──────▶ │ Effort model │──▶ hours needed ─────────┴─▶ "can we file in time?"
                   └──────────────┘
```

- The **API** uses the harm model at start-up to rank the whole queue.
- The **Incident detail** page shows all three outputs.
- The **LangGraph agents** receive these numbers as fixed facts and are told never to recalculate them.

⬅ Previous: [Data preprocessing](../data_pipeline/README.md) · ➡ Next: [RAG / knowledge base](../knowledge/README.md)
