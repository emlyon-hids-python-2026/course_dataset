# Course Dataset: Hospital Emergency Department (France)

A small, synthetic-looking relational dataset of emergency-room visits in France (2022–2024), built for the Python programming course (course 04). It is deliberately **messy**: it is meant for practising data loading, cleaning, merging and analysis with pandas.

## Repository layout

```
course_dataset/
└── hospital_emergency/
    ├── Patient.xlsx        # 2,686 patients (one row per patient)
    ├── Visit_Part1.csv     # 4,000 visits
    ├── Visit_Part2.csv     # 4,000 visits
    ├── Visit_Part3.csv     # 4,747 visits
    └── Department.csv      # 101 French départements and their regions
```

In total there are **12,747 visits** (the three visit files share one schema and must be concatenated).

## Entity relationships

```
Department (dept_code) 1 ───< Patient (dept_code) 1 ───< Visit (patient_id)
```

| Join | Left key | Right key | Notes |
|------|----------|-----------|-------|
| Visit → Patient | `Visit.patient_id` | `Patient.patientID` | Every visit matches a patient (no orphans) |
| Patient → Department | `Patient.dept_code` | `Department.Department_Code` | Codes look like `dep_75` |

Note that the key columns are named differently across files (`patient_id` vs `patientID`, `dept_code` vs `Department_Code`).

## File descriptions

### `Visit_Part1.csv`, `Visit_Part2.csv`, `Visit_Part3.csv`

One row per emergency visit. Date range: 2022-01-01 to 2024-12-31.

| Column | Description |
|--------|-------------|
| `patient_id` | Patient identifier. Mixed format: mostly 5–6 digit numbers, but about 20% are alphanumeric codes such as `jdqam4` |
| `arrival_datetime` | Arrival timestamp, `YYYY-MM-DD HH:MM` |
| `visit_id` | Unique visit ID, `URG-<patient_id>-<seq>` (unique across all three files) |
| `motif_category` | Reason for visit (see below) |
| `triage_level` | Triage severity, 1 (most urgent) to 5 (least urgent) |
| `wait_minutes` | Waiting time before care, in minutes |
| `act_unit_cost` | Unit cost of a medical act, in euros; fixed per category |
| `nb_acts` | Number of medical acts performed (1–4) |

A visit's total cost is `act_unit_cost * nb_acts`.

Reason-for-visit categories, with unit cost:

| Category | Unit cost (€) |
|----------|--------------|
| Trauma | 74 |
| Minor Injury | 45 |
| Fever / Infection | 52 |
| Abdominal Pain | 68 |
| Chest Pain | 98 |
| Respiratory Distress | 85 |

### `Patient.xlsx` (sheet `Sheet1`)

One row per patient, 2,686 unique IDs.

| Column | Description |
|--------|-------------|
| `patientID` | Patient identifier (same mixed numeric/alphanumeric format as in visits) |
| `sex` | `F` / `M` |
| `birth_date` | Birth date stored as an **Excel serial number** (e.g. `40532`), not a date |
| `dept_code` | Département of residence (17 distinct values, mostly Île-de-France, e.g. `dep_75`) |
| `city` | City of residence (29 distinct values) |
| `insurance_type` | `Social Security`, `Social Security + Private`, or `AME` (state medical aid) |

### `Department.csv`

Reference table of the 101 French départements (including overseas).

| Column | Description |
|--------|-------------|
| `Region_Name` | Region name (18 regions), with accents, e.g. `Île-de-France` |
| `Region_Code` | Numeric INSEE region code |
| `Department_Name` | Département name |
| `Department_Code` | Code prefixed with `dep_`, e.g. `dep_01` |

## Known data-quality issues (intentional)

These are the cleaning exercises built into the data:

- **Split visit table**: visits are spread over three files that must be concatenated.
- **Inconsistent category labels**: the same reason for visit is spelled several ways, e.g. `Abdominal Pain` / `Abd. Pain`, `Chest Pain` / `Ch. Pain`, `Fever / Infection` / `Fever/Inf.`, `Respiratory Distress` / `Resp. Distress`. Roughly 5% of rows use abbreviations.
- **Mixed-type IDs**: `patient_id` mixes numbers and alphanumeric strings. Read it as a string, otherwise IDs may be mangled or fail to join.
- **Excel serial dates**: `Patient.birth_date` must be converted (`origin="1899-12-30"` with `pd.to_datetime(..., unit="D")`) before computing age.
- **Missing values** in `Patient.xlsx`: `sex` (~286 blank), `city` (~1,157 blank), `insurance_type` (~75 blank).
- **Suspicious `wait_minutes`**: `Visit_Part1.csv` contains waits as low as 1 minute, while the other two files start at 10. Check the distribution for outliers.
- **Column-name mismatches** between files (see the join table above).

## Getting started

```python
import pandas as pd

base = "hospital_emergency/"

visits = pd.concat(
    [pd.read_csv(f"{base}Visit_Part{i}.csv", dtype={"patient_id": str}) for i in (1, 2, 3)],
    ignore_index=True,
)
patients = pd.read_excel(f"{base}Patient.xlsx", dtype={"patientID": str})
departments = pd.read_csv(base + "Department.csv")

# Clean
visits["motif_category"] = visits["motif_category"].replace({
    "Abd. Pain": "Abdominal Pain",
    "Ch. Pain": "Chest Pain",
    "Fever/Inf.": "Fever / Infection",
    "Resp. Distress": "Respiratory Distress",
})
visits["arrival_datetime"] = pd.to_datetime(visits["arrival_datetime"])
visits["total_cost"] = visits["act_unit_cost"] * visits["nb_acts"]
patients["birth_date"] = pd.to_datetime(patients["birth_date"], unit="D", origin="1899-12-30")

# Merge
df = (
    visits.merge(patients, left_on="patient_id", right_on="patientID", how="left")
          .merge(departments, left_on="dept_code", right_on="Department_Code", how="left")
)
```

Requires `pandas` and `openpyxl` (for reading `.xlsx`).

## Example questions to explore

- Average waiting time by triage level, reason for visit, or hour of day.
- Visit volume and revenue trends by month and year.
- Patient age distribution and its relation to reason for visit.
- Cost per visit by insurance type.
- Visits by region of residence.
- Patients with repeated visits.
