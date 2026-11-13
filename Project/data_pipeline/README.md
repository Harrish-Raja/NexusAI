# Data Preprocessing (`data_pipeline/`)

> **In one sentence:** this part takes five messy CSV files exported from the security desk's systems, repairs every kind of mess in them, and writes five clean CSV files that the machine-learning models can learn from.

Run it with:

```bash
python run_data_pipeline.py
```

---

## 1. Why do we need this at all?

Real exports are messy. The same fact is written in many different ways, and a model cannot learn from that. Some examples straight from the raw files:

| Problem | What the raw data looks like | What we want |
|---|---|---|
| Same city, many spellings | `Bangalore`, `BLR`, ` bengaluru ` | `Bengaluru` |
| "Missing" written many ways | `NA`, `n/a`, `-`, `?`, `Not recorded`, `null` | an empty cell |
| Yes/No written many ways | `Y`, `true`, `1`, `yes`, `T` | `Yes` / `No` |
| Timestamps in 4 notations and many time zones | `2026-03-12T11:40:00Z`, `12/03/2026 11:40 +02:00`, `12-Mar-2026 11:40 CET`, `1773315600` | one moment in UTC |
| Counts written like text | `1.2 lakh`, `12k`, `1,200` | `120000`, `12000`, `1200` |
| Money with a currency sign | `€1,200`, `1200 EUR` | `1200.0` |
| Durations with units | `2h 30m`, `45 min`, `3 h` | minutes (or hours) as a number |
| Impossible values | `systems_affected_count = 9000` | treated as missing |
| Timeline backwards | confirmed **before** the first alert | the two times swapped |
| Duplicate rows | the same row twice | kept once |
| No answer (target) | the label column is empty | row removed (can't learn from it) |
| Dirty free text | `&nbsp;`, `<b>…</b>`, ALL CAPS NOTES | plain, tidy text |

---

## 2. The five input files and what comes out

| Raw file (in `data/`) | Rows in | Rows kept | Used to train |
|---|---|---|---|
| `security_incidents_train.csv` | 30,728 | **29,715** | Harm model |
| `security_incidents_eval.csv` | 8,613 | **8,308** | Harm model (testing) |
| `containment_effort_train.csv` | 29,710 | **28,611** | Effort model |
| `containment_effort_eval.csv` | 6,132 | **5,913** | Effort model (testing) |
| `analyst_incident_notes.csv` | 34,685 | **32,405** | Attack-type model |

Outputs:

- `processed/*_clean.csv` → the five cleaned tables
- `output/data_quality_report.json` → a report saying *how dirty* each file was (duplicates, missing cells per column, out-of-range values, timestamp notations seen, etc.)

> The live queue files in `data/input/` (`open_incidents.csv`, `open_incident_notes.csv`) are **not** cleaned by this pipeline. They are already in clean form and are read directly by the API.

---

## 3. The files in this folder

| File | Job |
|---|---|
| `dataset_loader.py` | The **only** file that reads/writes CSVs. Raw files are read with *every column as text* so nothing is guessed wrongly on load. |
| `common_cleaning.py` | A toolbox of small repair functions shared by all datasets (strip spaces, fix missing tokens, parse timestamps, read counts/money/durations, range checks, fill gaps…). Every function returns a **new** table and never changes the one it was given. |
| `incident_cleaning.py` | The full cleaning recipe for the two **incident** files. |
| `effort_cleaning.py` | The full cleaning recipe for the two **containment effort** files. |
| `incident_note_cleaning.py` | The full cleaning recipe for the **analyst notes** (free text). |
| `quality_profiler.py` | Measures the mess in each raw file and writes the JSON quality report. |
| `feature_builder.py` | Not cleaning – it turns a clean table into model inputs (used by the ML part, see below). |

All the rules (valid values, spelling maps, range limits, which columns are which) live in `config/config.py`, not in the code. That means the cleaning logic is generic and the business knowledge sits in one place.

---

## 4. Step by step: how one incident file is cleaned

`incident_cleaning.clean_incident_dataset()` runs these steps **in this order** (the order matters):

```
 raw CSV (all text)
   │
   ├─ 1. Strip spaces              "  Lisbon "        → "Lisbon"
   ├─ 2. Missing tokens → empty    "N/A", "-", "?"    → (empty)
   ├─ 3. Fix categories            "Bangalore","BLR"  → "Bengaluru"
   │        then anything not in the allowed list     → (empty)
   ├─ 4. Fix Yes/No flags          "y","1","TRUE"     → "Yes"
   ├─ 5. Parse timestamps to UTC   "12-Mar-2026 11:40 CET" → 2026-03-12 10:40 UTC
   ├─ 6. Put timeline in order     confirmed < first alert → swap them
   ├─ 7. Counts / money / durations to numbers
   ├─ 8. Everything else numeric   "abc"              → (empty)
   ├─ 9. Out-of-range → empty      e.g. outage > 20,000 min
   ├─10. Recompute the interval    minutes from first alert → confirmation
   │
   ├─11. Drop rows with no answer  (high_risk_to_individuals is empty)
   ├─12. Drop exact duplicates
   │
   ├─13. Fill gaps:
   │       numbers     → the column's median
   │       timestamps  → rebuilt from the other timestamp + interval
   │       categories  → "Unknown"
   ├─14. Target Yes/No → 1/0
   └─15. Timestamps written back as text "2026-03-12T10:40:00Z"
   ▼
 processed/security_incidents_train_clean.csv
```

### Why this order?

- **Fix spellings before throwing away unknown values** (step 3) – otherwise `Bangalore` would be thrown away instead of corrected.
- **Range-check before filling gaps** (9 before 13) – an impossible value becomes a gap, and then gets the median like any other gap. If we filled first, the impossible values would distort the median.
- **Drop rows with no answer before filling** (11 before 13) – there is no point repairing a row we are about to delete.
- **Recompute the interval from the timestamps** (10) – the raw "minutes to confirmation" column often disagreed with the two times it was supposed to be computed from. The times are trusted; the stored number is recalculated.

### Timestamps in more detail

The raw files mix four notations, and the hubs are in four time zones (Bengaluru, Lisbon, Warsaw, Tallinn). `_parse_one_timestamp` handles:

1. **Epoch seconds** – `1773315600` → read as seconds since 1970, UTC.
2. **ISO with `Z`** – `…Z` is turned into `+00:00`.
3. **Named zones** – `CET`, `IST`, `EEST`… are swapped for their numeric offset using `TIMEZONE_ABBREVIATION_OFFSETS` (e.g. `IST` → `+05:30`).
4. **Numeric offsets** – tried against the three allowed formats.

Everything ends up in **UTC**, so deadlines can be compared fairly no matter which hub recorded them.

---

## 5. What's different for the other two datasets

### Containment effort (`effort_cleaning.py`)

Target column: `containment_hours` (how many analyst hours containment took).

- Durations are converted to **hours** (a bare number is already hours; `90 min` → `1.5`).
- **Forensic image fix:** if `forensic_image_required_flag = No` but `forensic_images_taken_count > 0`, the count is set to `0`. The flag is the decision; the count was a data-entry mistake.
- Missing dates are filled with the most common date.

### Analyst notes (`incident_note_cleaning.py`)

Target column: `attack_type` (one of 8 attack types). The important column is the free text `note_text`.

`repair_note_text()` does:
1. Replace HTML artefacts like `&nbsp;`, `&amp;` with the real character.
2. Remove markup tags like `<b>` / `<br>`.
3. Squash repeated spaces.
4. If a note is written **ENTIRELY IN CAPITALS**, lower-case it (shouting shouldn't change what the model sees).

Then:
- Attack type spellings are folded (`ddos`, `DoS`, `denial-of-service` → `Denial of service`).
- Notes with **fewer than 6 words** are removed – too short to say anything.
- `note_length_chars` is recomputed from the cleaned text.

---

## 6. The quality report

`quality_profiler.py` compares each raw file with its cleaned version and writes `output/data_quality_report.json`. For every dataset you get:

- `rows_received`, `rows_kept`, `rows_removed`
- `duplicate_rows`, `rows_with_no_answer`
- `missing_cells_by_column`, `out_of_range_by_column`
- `missing_cells_after_cleaning` (should be 0)

Plus extras per dataset – e.g. how many timestamps were in each notation, how many timelines were backwards, the share of high-risk incidents, the median containment hours, the share of each attack type.

This is the *evidence* that cleaning worked, and it's what you'd show someone who asks "how dirty was the data?".

---

## 7. `feature_builder.py` – the bridge to the models

After cleaning, a table still isn't ready for scikit-learn. `feature_builder.py` prepares it:

1. **`build_model_inputs`** – splits the table into `X` (inputs) and `y` (answer). It **drops identifier columns** such as `incident_id`, `incident_reference`, `campaign_reference` and the raw timestamps. These are unique per row; a model would just memorise them instead of learning real patterns.
2. **`split_column_kinds`** – works out which columns are numbers and which are categories.
3. **`build_native_category_pipeline`** (used by the harm and effort models) – builds a scikit-learn `Pipeline`:
   - numbers → median imputer
   - categories → fill with `"Unknown"` → `OrdinalEncoder` (category → integer code; unseen categories become `-1`)
   - tells the gradient-boosting model which columns are categories, so it treats them as categories rather than as ordered numbers.
4. **`build_model_pipeline`** – the same idea but with one-hot encoding (available as an alternative).

Because the preparation is **inside** the pipeline, the exact same preparation is applied when training *and* when scoring a live incident later. That prevents the classic "trained on one format, predicted on another" bug.

➡ Next: [Machine learning models](../modeling/README.md)
