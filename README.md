# Wind Turbine Anomaly Detection: an end-to-end pipeline on CARE to Compare

> **Status: Phase 0, understand the data.** No production code is written until the data audit (see "Phase 0 checklist") is finished and its findings are recorded in `docs/data_findings.md`.

## 1. Purpose

This is a learning and portfolio project, open source, with a write-up on <https://newtonandnetworks.com>.

Goals:
- Build a realistic end-to-end anomaly detection pipeline on multi-sensor industrial data: ingestion, local SQL database, cleaning, analysis, modelling, evaluation, deployment simulation.
- Compare several anomaly detection methods fairly, using event-level metrics that match how an operator uses an alarm (not just point-wise precision/recall).
- Practise data processing, SQL, statistical comparison of methods (A/B-style paired tests), and evaluation.
- End with production-quality code: tests, config, logging, reproducible runs.

Non-goals (for now): live deployment, vibration or high-frequency analysis, real-time serving.

**Scope rule:** use only the public dataset below and public literature. Do not use any confidential data, code or knowledge from previous employment.

## 2. Instructions for Claude (read first)

1. **Do not assume facts about the data.** Section 4 separates facts from the paper from things that still need checking against the actual files. Verify against the files and record the result in `docs/data_findings.md`, with the query or code used.
2. **Never invent numbers.** If a value is unknown, say so and propose how to measure it.
3. **Headline metrics are event-level**: the CARE score and its four sub-scores, false alarms per turbine-month, and lead time before the fault. Point-wise F1 may be reported as a secondary metric only.
4. **Split by turbine, never by file.** The same turbine appears in several datasets, so a file-level split leaks information.
5. **Respect the status IDs.** Training data is filtered to normal status. Rows with abnormal status are excluded from scoring.
6. **Pin the dataset version** (see 3) and record its checksum. Baseline numbers from the literature are only comparable on the same version.
7. Prefer small, tested, reproducible steps. Notebooks are for exploration; reusable logic goes in the package.
8. Flag anything that looks inconsistent with this README instead of silently working around it.

## 3. Dataset

| Item | Value |
|---|---|
| Name | CARE to Compare, wind turbine SCADA anomaly detection |
| Authors | Christian Gück, Cyriana M.A. Roelofs, Stefan Faulstich (Fraunhofer IEE) |
| Paper | arXiv:2404.10320 |
| Zenodo | concept DOI 10.5281/zenodo.10958774; version 6 DOI 10.5281/zenodo.15846963; record 10958775 lists a ~5.5 GB zip (`CARE_To_Compare_Data.zip`) |
| Licence | CC BY-SA 4.0. Any redistributed derived version of the data must use the same licence. Cite the paper. |
| Version to use | **TBD, decide in Phase 0** (v6 was used in a 2026 follow-up paper). Record the version, download date and md5. |

Note: the paper (April 2024) reports 44 anomaly datasets; the Zenodo record text I saw says 45 and a re-upload says 44. Counts may differ by version, so recount from the files.

## 4. What we know about the data (from the paper)

### 4.1 Structure

| | Farm A | Farm B | Farm C | Overall |
|---|---|---|---|---|
| Location | Onshore, Portugal (based on EDP open data) | Offshore, Germany | Offshore, Germany | |
| Turbines | 5 | 9 | 22 | 36 |
| Datasets (CSV files) | 22 | 15 | 58 | 95 |
| Anomaly datasets | 11 | 6 | 27 | 44 |
| Normal datasets | 11 | 9 | 31 | 51 |
| Sensors | 54 | 63 | 238 | |
| Feature columns | 86 | 257 | 957 | |

- Resolution: 10 minutes. About 89 years of data in total.
- "Datasets" are sub-datasets, each one turbine's time series. They are **not** distinct turbines, so the same turbine can appear in several files. Asset IDs are assigned so that datasets of the same turbine can be linked.
- Each sensor has a 10-minute average; some also have min, max and standard deviation columns.
- Each file has 5 descriptive columns: row ID, timestamp, asset ID, train/test flag, status ID.
- Only power, reactive power and wind speed have recognisable column names. Power and reactive power are scaled by rated power. Other sensors are anonymised, with separate descriptions (text, unit, whether regular signal, counter or angle).
- Timestamps were shifted by a random number of whole years per dataset. This preserves seasonality but distorts the order between datasets.

### 4.2 Train / prediction design

- Each file has **one year of training data** plus **4 to 98 days of prediction data**.
- Labeled anomaly events lie inside the prediction part only. The prediction part contains the event time frame with padding before and after, so a model cannot guess the label from file length.
- An anomaly dataset is mostly normal data with one labeled event inside the prediction part. It does not mean the whole file is anomalous.
- The event has a start timestamp, and its end is the start of the turbine fault.
- Training requirement: at least 2/3 of the training data must be normal by status ID. The rest may contain abnormal states and must be filtered.

### 4.3 Labels

Dataset level: anomaly or normal. Event level: start timestamp (end = fault start). Timestamp level: status ID.

| Status ID | Description | Considered normal |
|---|---|---|
| 0 | Normal operation without limitations | Yes |
| 1 | Derated power generation (power restriction) | No |
| 2 | Idling, waits to operate again | Yes |
| 3 | Service mode / service team on site | No |
| 4 | Down due to a fault or other reasons | No |
| 5 | Other states (system test, setup, ice build-up, emergency power) | No |

How labels were made (per the paper):
- Anomaly labels: operator feedback, or documented faults in service reports and fault logbooks. Normal labels: operator feedback, manual inspection, expert knowledge.
- Farm A: the EDP logbook only has fault start timestamps. Event starts were chosen by analysing data before each fault, so true starts can differ. Status IDs were derived from the logbook: 14 days before each fault marked 4, 3 days after marked 3.
- Farms B and C: starts defined from data analysis, operator feedback, service reports and expert knowledge. The authors say the start is unlikely to be too early, but could be earlier than defined. Status IDs come from operator operating modes plus service reports.
- Ground truth cannot be perfect: the onset of anomalous behaviour is inferred, not observed.

### 4.4 Known data issues

- Farms B and C use 0 in place of missing values, so long runs of zeros need care.
- Status for B and C can be inconsistent (often logged only on change; a brief communication error can break it). Farm A's status is derived from the fault logbook only. Cross-check with power and wind speed.
- The Zenodo record has a "Known Data Issues" section that appears to list issues involving grid reactive power, pitch angle and specific sensors. Read it fully in Phase 0.

## 5. Evaluation: the CARE score

From the paper (verify against the reference implementation, see Phase 0):

- **Coverage**: F-beta (beta = 0.5) per anomaly dataset, on rows with normal status only.
- **Accuracy**: TN / (FP + TN) per normal dataset, on rows with normal status only.
- **Reliability**: event-wise F-beta (beta = 0.5). A prediction counts as an alarm if a "criticality" counter reaches 72. The counter increases by 1 per detected row with normal status and decreases by 1 (not below 0) per undetected normal-status row. 72 steps = 12 hours of consecutive detections.
- **Earliness**: weighted score over the event; weight 1 for the first half of the event, then decreasing linearly to 0 at the end.
- Combination: weighted average with weights 1 (coverage), 1 (earliness), 1 (reliability), 2 (accuracy). Special cases: CARE = 0 if no anomaly is detected; CARE = Accuracy if Accuracy < 0.5.

Reference numbers (paper's mini-benchmark): autoencoder 0.66, isolation forest 0.14, random 0.50, all-normal and all-anomaly 0. A 2026 paper reports PCA monitoring at 0.5773 on v6. **Re-run all baselines ourselves on the pinned version.**

Additional operational metrics we want:
- False alarms per turbine-month on normal data.
- Lead time (time from first alarm to the fault start).
- Sensitivity to anomaly prevalence. The dataset is balanced by design (44 of 95), while real faults are rare, so report how precision degrades when anomaly datasets are subsampled.

## 6. Phase 0 checklist (data audit)

Answer each item from the files and write the answer, with evidence, to `docs/data_findings.md`.

1. Folder layout, file naming, and which files belong to which farm. Is there a metadata or event-info file with event start/end timestamps and fault descriptions?
2. Row counts, column counts and dtypes per file. Do they match 4.1?
3. Is the prediction part contiguous and after the training part? Verify the train/test column against timestamps. Any gaps in the 10-minute grid?
4. Event durations (minutes) per anomaly dataset, padding before and after, and total prediction length. Distribution by farm.
5. Status ID distribution in training vs prediction parts, per farm. How much training data survives the "normal status" filter?
6. Zero runs in farms B and C: where, how long, which sensors.
7. Which turbines appear in several datasets. Build the turbine-level split.
8. Anomaly vs normal counts per farm for the chosen version.
9. Sensor description file: which sensors are counters or angles, and units.
10. The Known Data Issues section, and the affected sensors and periods.
11. Locate or reimplement the reference CARE-score code and reproduce the paper's baselines (at least random, all-normal, all-anomaly) as a sanity check.

## 7. Roadmap

| Phase | Goal | Output |
|---|---|---|
| 0 | Data audit | `docs/data_findings.md`, pinned version + checksum |
| 1 | Local database | Schema, loader, row-count checks (DuckDB or PostgreSQL, decide here) |
| 2 | Cleaning | Missing-value handling, status filtering, plausibility rules (power vs wind speed) |
| 3 | Baselines | Trivial strategies, PCA residuals, isolation forest; CARE scoring working |
| 4 | Models | Normal-behaviour models (regression, autoencoder), alarm logic with minimum duration |
| 5 | Evaluation | Paired comparison across turbines, prevalence sensitivity, per-farm analysis, cost assumptions clearly labelled |
| 6 | Deployment simulation | Replay each prediction part in time order, report alarms and lead time |
| 7 | Productionise | Package, config, tests, CI, logging, reproducible runs, data versioning |
| 8 | Write-up | Blog post with the data problems, results and honest limitations |

## 8. Proposed repository layout (adjust in Phase 1)

```
.
├── README.md
├── docs/
│   └── data_findings.md        # Phase 0 results
├── data/                       # gitignored; raw files, never committed
├── src/windad/
│   ├── ingest/                 # loaders, schema
│   ├── clean/
│   ├── features/
│   ├── models/
│   ├── scoring/                # CARE score + operational metrics
│   └── simulate/               # time-ordered replay
├── sql/                        # schema and queries
├── notebooks/                  # exploration only
├── tests/
└── pyproject.toml
```

## 9. Open decisions

- Dataset version to pin.
- Database: DuckDB (simple, local, analytical) vs PostgreSQL.
- Code licence (the data stays CC BY-SA 4.0).
- Whether to add Kelmarsh or Penmanshiel (Cubico, open SCADA + event logs) as a second, unlabeled stream. They have event logs but no curated anomaly labels, so any labels would be weak labels we derive ourselves.
- Guard gap before the labeled event start when building a "healthy" reference, chosen from the data.

## 10. Citation

Gück, C., Roelofs, C.M.A., Faulstich, S. (2024). CARE to Compare: A real-world dataset for anomaly detection in wind turbine data. arXiv:2404.10320.
