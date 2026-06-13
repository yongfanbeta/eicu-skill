# eicu-skill 🏥

> AI Skill for querying the **eICU Collaborative Research Database 2.0**

An AI agent skill that helps query the [eICU 2.0](https://eicu-crd.mit.edu/) critical care database. When a user asks for ICU patient data — vital signs, lab results, vasopressors, GCS, diagnoses — the agent reads the skill's reference docs and generates ready-to-run SQL or Python code tailored to the request.

## How It Works

```
User asks: "从 eICU 提取脓毒症患者第一天的生命体征和乳酸"
        ↓
Agent loads eicu-skill → reads relevant reference files
        ↓
Agent generates custom SQL/Python code with correct table names, field mappings, and filters
        ↓
User runs the code against their own PostgreSQL database
```

**This skill does NOT connect to any database.** It provides the knowledge (schema, field mappings, query patterns) so that an AI agent can write correct queries. Users run the queries themselves against their own eICU instance.

## What It Generates

Depending on the user's request, the agent will produce:

| Output | When | Example |
|--------|------|---------|
| **SQL queries** | User wants to run directly in psql / pgAdmin / DBeaver | `SELECT ... FROM vitalperiodic WHERE patientunitstayid = ...` |
| **Python scripts** | User wants programmatic access via psycopg2 + pandas | `extract_vital_signs(patient_ids=[...])` |
| **Pivot tables** | User needs wide-format data for analysis | Scripts that reshape nursecharting → one-row-per-patient tables |
| **Data extraction pipeline** | Complex multi-step requests | Combined scripts that join vitals + labs + GCS + vasopressors into a flat table |

## Installation

### Option 1: Clone into your AI agent's skills/knowledge directory

```bash
git clone https://github.com/yongfanbeta/eicu-skill.git <your-agent-skills-dir>/eicu-skill
```

### Option 2: Via SkillHub (for OpenClaw users)

```bash
openclaw skill install eicu-skill
```

After installation, the skill is automatically available in your AI agent. No additional configuration needed.

## File Structure

```
eicu-skill/
├── SKILL.md                        # Core skill definition & usage guide
├── README.md                       # This file
├── LICENSE                         # MIT
└── references/
    ├── schema.md                   # Full database schema, all tables & relationships
    ├── demographics.md             # Patient demographics, hospital info, ICU type
    ├── vital_signs.md              # vitalperiodic & vitalaperiodic tables
    ├── vital_signs_nursecharting.md # nursecharting table (most detailed vital signs source)
    ├── labs.md                     # Laboratory results (labname → value mapping)
    ├── blood_gas.md                # Arterial/venous blood gas analysis
    ├── gcs.md                      # Glasgow Coma Scale extraction from nursecharting
    ├── vasopressors.md             # Vasopressor identification & dosing
    ├── infusions.md                # Continuous infusion drugs
    ├── medications.md              # Medication orders
    ├── urine_output.md             # Urine output measurement
    ├── oxygenation.md              # Oxygen delivery (FiO2, O2 device, SpO2/FiO2)
    ├── weight.md                   # Patient weight (admission/daily)
    ├── diagnoses.md                # APACHE admission diagnosis & grouping
    └── common_queries.md           # Pre-built: first-day vitals, SOFA, sepsis, mortality
```

## What's Inside

### Key References

| File | What It Covers |
|------|---------------|
| `schema.md` | All eICU tables, field definitions, row counts |
| `vital_signs.md` | `vitalperiodic` (periodic) & `vitalaperiodic` (aperiodic) tables |
| `vital_signs_nursecharting.md` | `nursecharting` table — most detailed source, requires pivoting |
| `labs.md` | `lab` table with `labname` → value mapping (no itemid needed!) |
| `blood_gas.md` | Blood gas results: pH, pO2, pCO2, lactate, etc. |
| `gcs.md` | GCS Total + Motor/Verbal/Eyes sub-scores from nursecharting |
| `vasopressors.md` | Norepinephrine, vasopressin, phenylephrine, dopamine, etc. |
| `diagnoses.md` | APACHE IV admission diagnosis grouping (Sepsis, ACS, CHF, PNA...) |
| `common_queries.md` | ICU stay detail, first-day pivoted vitals/labs, SOFA, mortality |

### eICU vs MIMIC-IV Key Differences

| Feature | eICU 2.0 | MIMIC-IV |
|---------|----------|----------|
| Time representation | **Offset in minutes** (relative to ICU admit) | Absolute timestamps (UTC) |
| Vital signs source | 3 tables: `vitalperiodic`, `vitalaperiodic`, `nursecharting` | `chartevents` (single table) |
| Lab results | `labname` is the test name directly | `itemid` → `d_labitems` mapping needed |
| Patient ID | `patientunitstayid` | `stay_id` |
| Field naming | camelCase-ish lowercase (`heartrate`, `sao2`) | snake_case (`heart_rate`, `spo2`) |
| Dataset size | ~200K patients, ~200 hospitals | ~50K patients, single hospital |

## Usage Examples

After installing, just talk to your OpenClaw agent naturally:

| What You Say | What The Agent Does |
|-------------|-------------------|
| "从 eICU 提取第一天生命体征" | Reads `vital_signs.md`, generates query for `vitalperiodic` |
| "eICU 里怎么提取 GCS 评分？" | Reads `gcs.md`, generates nursecharting pivot query |
| "帮我提取升压药使用情况" | Reads `vasopressors.md`, generates vasopressor extraction |
| "eICU 的实验室检查怎么查？" | Reads `labs.md`, generates lab query with `labname` filter |
| "提取尿量和体重数据" | Reads `urine_output.md` + `weight.md`, generates combined script |
| "怎么把 nursecharting 转成宽表？" | Reads `vital_signs_nursecharting.md`, generates pivot query |

## Important Notes

- **No database access**: This skill only provides query knowledge. You need your own eICU PostgreSQL instance.
- **Access required**: Apply for eICU access at [PhysioNet](https://physionet.org/content/eicu-crd/2.0/).
- **Time offsets**: All times are in **minutes relative to ICU admission** (`offset = 0` = ICU admit, `offset = 1440` = 24h later).
- **Data quality**: eICU has more missing values and outliers than MIMIC-IV. Always add range checks (e.g., `heartrate BETWEEN 25 AND 225`).
- **Text values**: Some fields contain non-numeric text (e.g., `'80s'` for age). The skill documents how to handle these.
- **Big tables**: `nursecharting` (~151M rows), `vitalperiodic` (~147M rows), `lab` (~39M rows). Always filter by `patientunitstayid` + time range.

## References

- [eICU Data Documentation](https://eicu-crd.mit.edu/)
- [eICU SchemaSpy](https://lcp.mit.edu/eicu-schema-spy/index.html)
- [eICU on PhysioNet](https://physionet.org/content/eicu-crd/2.0/)
- [eicu-code Repository](https://github.com/MIT-LCP/eicu-code)

## License

MIT
