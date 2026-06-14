---
name: eicu-skill
description: Query the eICU Collaborative Research Database 2.0. Generates SQL and Python code for extracting ICU data including vital signs, labs, GCS, vasopressors, blood gas, oxygenation, and diagnoses from PostgreSQL. Supports both direct SQL queries and Python (psycopg2) scripts.
---

# eICU 2.0 Data Extraction Skill

> **⚠️ Important Correction (2026-05-20)**: This document has been thoroughly revised based on the **official SchemaSpy documentation** and **MIT-LCP/eicu-code repository**.
>
> **Key Corrections**:
> - ✅ Both `vitalperiodic` and `vitalaperiodic` tables **exist** (I previously incorrectly believed they did not)
> - ✅ Field names use **camelCase** (`heartrate` not `heart_rate` or `heartRate`)
> - ✅ Vital signs data comes from **multiple tables** (`vitalperiodic`, `vitalaperiodic`, `nursecharting`)

Extract critical care data from the eICU Collaborative Research Database 2.0. Provides SQL templates and Python code; users handle database connections independently.

## Connection Method

Users provide their own PostgreSQL connection parameters. This skill only provides query code.

## Supported Query Types

| Query Type | Reference File |
|-----------|----------|
| Patient Basic Information | [references/schema.md](references/schema.md) (patient table) |
| Vital Signs (vitalperiodic) | [references/vital_signs.md](references/vital_signs.md) |
| Vital Signs (vitalaperiodic) | [references/vital_signs.md](references/vital_signs.md) |
| Vital Signs (nursecharting) | [references/vital_signs_nursecharting.md](references/vital_signs_nursecharting.md) |
| Physical Examination (GCS, etc.) | [references/gcs.md](references/gcs.md) |
| Laboratory Tests | [references/labs.md](references/labs.md) |
| Blood Gas Analysis | [references/blood_gas.md](references/blood_gas.md) |
| Urine Output | [references/urine_output.md](references/urine_output.md) |
| Vasopressors | [references/vasopressors.md](references/vasopressors.md) |
| Infusions | [references/infusions.md](references/infusions.md) |
| Weight | [references/weight.md](references/weight.md) |
| Oxygenation | [references/oxygenation.md](references/oxygenation.md) |
| Diagnoses and APACHE Grouping | [references/diagnoses.md](references/diagnoses.md) |
| Medications | [references/medications.md](references/medications.md) |
| Database Schema | [references/schema.md](references/schema.md) |
| Common Query Templates | [references/common_queries.md](references/common_queries.md) |

## Workflow

1. Confirm the query type needed by the user
2. Read the corresponding references file to get the SQL/Python template
3. Adjust query conditions based on the user's specific requirements
4. Provide both SQL and Python implementations

## Key Differences Between eICU and MIMIC-IV

| Feature | eICU 2.0 | MIMIC-IV |
|--------|----------|----------|
| **Time Representation** | Offsets (minutes) | Absolute timestamps |
| **Vital Signs Source** | `vitalperiodic` + `vitalaperiodic` + `nursecharting` | `chartevents` table (mapped via `d_items`) |
| **Laboratory Tests** | `lab` table (`labname` is the name directly) | `labevents` table (mapped via `d_labitems`) |
| **Patient ID** | `patientunitstayid` | `stay_id` |
| **Data Volume** | ~200,000 patients | ~50,000 patients |
| **Field Naming** | CamelCase (e.g., `heartrate`) | All lowercase, snake_case |

## Core Table Structures (Correct)

### 1. Patient Basic Information - `patient` Table

```sql
SELECT 
    patientunitstayid  -- ICU stay unique identifier
  , uniquepid           -- Patient unique identifier
  , patienthealthsystemstayid  -- Hospital stay identifier
  , hospitalid          -- Hospital ID
  , unitvisitnumber     -- ICU admission count
  , unittype            -- ICU type
  , apacheadmissiondx   -- APACHE IV admission diagnosis (important!)
  , age                 -- Age (censored: >89 displayed as 91)
  , gender              -- Gender ('Male'/'Female')
  , ethnicity           -- Ethnicity
  , hospitaladmitoffset  -- Hospital admission offset (minutes)
  , hospitaldischargeoffset  -- Hospital discharge offset (minutes)
  , unitdischargeoffset  -- ICU discharge offset (minutes)
  , hospitaldischargestatus  -- Hospital discharge status ('Alive'/'Expired')
  , admissionheight     -- Admission height (cm)
  , admissionweight     -- Admission weight (kg)
  , dischargeweight     -- Discharge weight (kg)
FROM patient;
```

**Calculating ICU Length of Stay (hours)**:
```sql
ROUND(unitdischargeoffset/60) AS icu_los_hours
```

### 2. Periodic Vital Signs - `vitalperiodic` Table

**Table Name**: `vitalperiodic`  
**Record Count**: 146,671,642 rows  
**Description**: Periodic vital sign measurements, typically recorded every 1-5 minutes.

```sql
SELECT 
    vitalperiodicid     -- Record ID (PK)
  , patientunitstayid  -- Patient ID (FK)
  , observationoffset   -- Observation offset (minutes, relative to ICU admission)
  , temperature         -- Temperature (°C)
  , sao2               -- Oxygen saturation (%)
  , heartrate          -- Heart rate (bpm)
  , respiration        -- Respiratory rate (breaths/min)
  , cvp                -- Central venous pressure (mmHg)
  , etco2              -- End-tidal CO2 (mmHg)
  , systemicsystolic   -- Invasive systolic blood pressure (mmHg)
  , systemicdiastolic  -- Invasive diastolic blood pressure (mmHg)
  , systemicmean       -- Invasive mean arterial pressure (mmHg)
  , pasystolic         -- Pulmonary artery systolic pressure (mmHg)
  , padiastolic        -- Pulmonary artery diastolic pressure (mmHg)
  , pamean             -- Pulmonary artery mean pressure (mmHg)
  , st1                -- ST segment lead 1 (mV)
  , st2                -- ST segment lead 2 (mV)
  , st3                -- ST segment lead 3 (mV)
  , icp                -- Intracranial pressure (mmHg)
FROM vitalperiodic
WHERE observationoffset >= 0 AND observationoffset < 1440  -- First 24 hours after admission
  AND patientunitstayid = %(patient_id)s
ORDER BY observationoffset;
```

**Notes**:
- All blood pressure values are invasive measurements (arterial line)
- `observationoffset` is minutes relative to ICU admission time
- Add plausibility checks: `heartrate BETWEEN 25 AND 225`

---

### 3. Aperiodic Vital Signs - `vitalaperiodic` Table

**Table Name**: `vitalaperiodic`  
**Record Count**: 25,075,074 rows  
**Description**: Aperiodic vital sign measurements (typically invasive monitoring data).

```sql
SELECT 
    vitalaperiodicid   -- Record ID (PK)
  , patientunitstayid   -- Patient ID (FK)
  , observationoffset    -- Observation offset (minutes)
  , noninvasivesystolic  -- Non-invasive systolic blood pressure (mmHg)
  , noninvasivediastolic -- Non-invasive diastolic blood pressure (mmHg)
  , noninvasivemean     -- Non-invasive mean arterial pressure (mmHg)
  , paop               -- Pulmonary artery occlusion pressure (mmHg)
  , cardiacoutput       -- Cardiac output (L/min)
  , cardiacinput        -- Cardiac index (L/min/m²)
  , svr                -- Systemic vascular resistance (dyn·s·cm⁻⁵)
  , svri               -- Systemic vascular resistance index (dyn·s·cm⁻⁵/m²)
  , pvr                -- Pulmonary vascular resistance (dyn·s·cm⁻⁵)
  , pvri               -- Pulmonary vascular resistance index (dyn·s·cm⁻⁵/m²)
FROM vitalaperiodic
WHERE observationoffset >= 0 AND observationoffset < 1440  -- First 24 hours after admission
  AND patientunitstayid = %(patient_id)s
ORDER BY observationoffset;
```

**Notes**:
- This table primarily contains invasive hemodynamic monitoring data
- `paop` (Pulmonary Artery Occlusion Pressure) is also known as PCWP (Pulmonary Capillary Wedge Pressure)
- `svr`/`svri` and `pvr`/`pvri` are used to assess vascular resistance

---

### 4. Vital Signs from Nursing Charts - `nursecharting` Table

**Table Name**: `nursecharting`  
**Record Count**: 151,604,232 rows  
**Description**: Nursing chart records (most detailed nursing data, including vital signs, scores, etc.).

```sql
SELECT 
    nursechartid              -- Record ID (PK)
  , patientunitstayid        -- Patient ID (FK)
  , nursingchartoffset       -- Charting offset time (minutes)
  , nursingchartentryoffset  -- Entry offset time (minutes)
  , nursingchartcelltypecat  -- Major category (e.g., 'Vital Signs')
  , nursingchartcelltypevallabel  -- Middle category (e.g., 'Heart Rate', 'Non-Invasive BP')
  , nursingchartcelltypevalname   -- Minor category (e.g., 'Heart Rate', 'Non-Invasive BP Systolic')
  , nursingchartvalue        -- Value (text or numeric)
FROM nursecharting
WHERE nursingchartcelltypecat IN ('Vital Signs', 'Scores', 'Other Vital Signs and Infusions')
  AND patientunitstayid = %(patient_id)s
  AND nursingchartoffset >= 0 AND nursingchartoffset < 1440  -- First 24 hours after admission
ORDER BY nursingchartoffset;
```

**Common Patterns for Extracting Vital Signs** (learned from eicu-code):

```sql
SELECT 
    patientunitstayid
  , nursingchartoffset
  , AVG(CASE WHEN nursingchartcelltypevallabel = 'Heart Rate'
               AND nursingchartcelltypevalname = 'Heart Rate'
               AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
               AND nursingchartvalue NOT IN ('-','.')
              THEN CAST(nursingchartvalue AS NUMERIC)
              ELSE NULL END) AS heartrate
  , AVG(CASE WHEN nursingchartcelltypevallabel = 'Non-Invasive BP'
               AND nursingchartcelltypevalname = 'Non-Invasive BP Systolic'
               -- similar handling for other vital signs...
              THEN CAST(nursingchartvalue AS NUMERIC)
              ELSE NULL END) AS nibp_systolic
FROM nursecharting
WHERE nursingchartcelltypecat IN ('Vital Signs', 'Scores', 'Other Vital Signs and Infusions')
  AND patientunitstayid = %(patient_id)s
  AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
GROUP BY patientunitstayid, nursingchartoffset
ORDER BY patientunitstayid, nursingchartoffset;
```

---

### 5. Laboratory Tests - `lab` Table

**Table Name**: `lab`  
**Record Count**: 39,132,531 rows  
**Description**: Laboratory test results. `labname` is the test name directly (**no itemid mapping needed**).

```sql
SELECT 
    labid                   -- Record ID (PK)
  , patientunitstayid       -- Patient ID (FK)
  , labresultoffset         -- Result offset time (minutes)
  , labresultrevisedoffset  -- Result revision offset time (minutes)
  , labname                 -- Test name (string)
  , labresult               -- Test result (numeric)
FROM lab
WHERE labname IN (
      'albumin'
    , 'total bilirubin'
    , 'BUN'
    , 'calcium'
    , 'chloride'
    , 'creatinine'
    , 'bedside glucose', 'glucose'
    , 'bicarbonate'  -- HCO3
    , 'Total CO2'
    , 'Hct'
    , 'Hgb'
    , 'PT - INR'
    , 'PTT'
    , 'lactate'
    , 'platelets x 1000'
    , 'potassium'
    , 'sodium'
    , 'WBC x 1000'
    , '-bands'
    -- Liver enzymes
    , 'ALT (SGPT)'
    , 'AST (SGOT)'
    , 'alkaline phos.'
  )
  AND labresult IS NOT NULL
  AND patientunitstayid = %(patient_id)s
  AND labresultoffset >= 0 AND labresultoffset < 1440  -- First 24 hours after admission
  -- Add plausibility checks
  AND (
        (labname = 'albumin' AND labresult >= 0.5 AND labresult <= 6.5)
    OR (labname = 'creatinine' AND labresult >= 0.1 AND labresult <= 28.28)
    -- ... plausibility checks for other tests
  )
ORDER BY labresultoffset;
```

**Common `labname` Values**:
- Albumin: `'albumin'`
- Total bilirubin: `'total bilirubin'`
- Blood urea nitrogen: `'BUN'`
- Calcium: `'calcium'`
- Chloride: `'chloride'`
- Creatinine: `'creatinine'`
- Blood glucose: `'bedside glucose'` or `'glucose'`
- Bicarbonate: `'bicarbonate'` or `'Total CO2'`
- Hematocrit: `'Hct'`
- Hemoglobin: `'Hgb'`
- Prothrombin time INR: `'PT - INR'`
- Partial thromboplastin time: `'PTT'`
- Lactate: `'lactate'`
- Platelets: `'platelets x 1000'`
- Potassium: `'potassium'`
- Sodium: `'sodium'`
- White blood cells: `'WBC x 1000'`
- Bands: `'-bands'`
- Alanine aminotransferase: `'ALT (SGPT)'`
- Aspartate aminotransferase: `'AST (SGOT)'`
- Alkaline phosphatase: `'alkaline phos.'`

---

### 6. Diagnoses - `patient` Table (APACHE Diagnoses)

Diagnosis information in eICU primarily comes from the `apacheadmissiondx` field in the `patient` table (APACHE IV admission diagnosis).

```sql
SELECT 
    patientunitstayid
  , apacheadmissiondx
  , age
  , gender
  , hospitaldischargestatus
  , ROUND(unitdischargeoffset/60) AS icu_los_hours
FROM patient
ORDER BY patientunitstayid;
```

**APACHE Diagnosis Grouping** (learned from eicu-code):

eicu-code provides `diagnosis/apache-groups.sql`, which groups `apacheadmissiondx` into:
- `'ACS'` (Acute Coronary Syndrome)
- `'ChestPainUnknown'` (Chest pain of unknown etiology)
- `'CHF'` (Congestive Heart Failure)
- `'CardiacArrest'` (Cardiac Arrest)
- `'CABG'` (Coronary Artery Bypass Graft)
- `'ValveDz'` (Valvular Heart Disease)
- `'PNA'` (Pneumonia)
- `'RespMedOther'` (Other Respiratory Diseases)
- `'Asthma-Emphys'` (Asthma/Emphysema)
- `'GIBleed'` (Gastrointestinal Bleeding)
- `'GIObstruction'` (Intestinal Obstruction)
- `'CVA'` (Cerebrovascular Accident)
- `'Neuro'` (Neurological Diseases)
- `'Coma'` (Coma)
- `'Overdose'` (Drug Overdose)
- `'Sepsis'` (Sepsis)
- `'ARF'` (Acute Renal Failure)
- `'DKA'` (Diabetic Ketoacidosis)
- `'Trauma'` (Trauma)
- `'Other'` (Other)

---

## Time Representation (Important!)

eICU uses **offsets (minutes)** to represent time, with all offsets relative to **ICU admission time**:

- `offset = 0`: ICU admission moment
- `offset = 60`: 60 minutes after admission (1 hour)
- `offset = 1440`: 1440 minutes after admission (24 hours)
- **Negative values**: Records before admission (if any)

**Key Time Fields**:
- `hospitaladmitoffset` - Hospital admission offset
- `hospitaldischargeoffset` - Hospital discharge offset
- `unitdischargeoffset` - ICU discharge offset (**used for calculating ICU LOS**)
- `observationoffset` - Vital signs record offset time
- `nursingchartoffset` - Nursing chart offset time
- `labresultoffset` - Laboratory result offset time
- `diagnosisoffset` - Diagnosis offset time

**Calculating ICU Length of Stay (hours)**:
```sql
ROUND(unitdischargeoffset/60) AS icu_los_hours
```

---

## Data Quality and Performance Optimization

### Data Quality

- **High Missing Rates**: Data completeness in eICU is lower than in MIMIC-IV
- **Outliers**: Always add plausibility checks in queries (e.g., `heartrate BETWEEN 25 AND 225`)
- **Text Values**: Some vital sign values are text (e.g., '80s' means 'in their 80s'), requiring cleaning

### Performance Optimization

**Large Tables** (from eicu-code statistics):
- `nursecharting`: ~151 million rows
- `lab`: ~39.13 million rows
- `vitalperiodic`: ~146.67 million rows
- `vitalaperiodic`: ~25.07 million rows
- `patient`: ~200,000 rows

**Optimization Tips**:
1. **Filter by `patientunitstayid`** (most important!)
2. **Limit time range** (`observationoffset BETWEEN 0 AND 1440`)
3. **Add `LIMIT` clause** (during testing)
4. **Use `EXPLAIN` to analyze query plans**
5. **Consider creating materialized views** (`CREATE MATERIALIZED VIEW`)

---

## Common Query Scenarios

### 1. Extract Patient Basic Information

```sql
SELECT 
    pt.patientunitstayid
  , pt.age
  , pt.apacheadmissiondx
  , CASE WHEN pt.gender = 'Male' THEN 1
         WHEN pt.gender = 'Female' THEN 2
         ELSE NULL END AS gender
  , CASE WHEN pt.hospitaldischargestatus = 'Alive' THEN 0
         WHEN pt.hospitaldischargestatus = 'Expired' THEN 1
         ELSE NULL END AS hosp_mortality
  , ROUND(pt.unitdischargeoffset/60) AS icu_los_hours
FROM eicu_crd.patient pt
ORDER BY pt.patientunitstayid;
```

### 2. Extract ICU Stay Details (Materialized View)

```sql
DROP MATERIALIZED VIEW IF EXISTS icustay_detail CASCADE;
CREATE MATERIALIZED VIEW icustay_detail AS (
  SELECT 
      pt.uniquepid
    , pt.patienthealthsystemstayid
    , pt.patientunitstayid
    , pt.unitvisitnumber
    , pt.hospitalid
    , h.region
    , pt.unittype
    , pt.hospitaladmitoffset
    , pt.hospitaldischargeoffset
    , 0 AS unitadmitoffset
    , pt.unitdischargeoffset
    , ap.apachescore AS apache_iv
    , pt.hospitaldischargeyear
    , pt.age
    , CASE WHEN LOWER(pt.hospitaldischargestatus) LIKE '%alive%' THEN 0
           WHEN LOWER(pt.hospitaldischargestatus) LIKE '%expired%' THEN 1
           ELSE NULL END AS hosp_mort
    , CASE WHEN LOWER(pt.gender) LIKE '%female%' THEN 0
           WHEN LOWER(pt.gender) LIKE '%male%' THEN 1
           ELSE NULL END AS gender
    , pt.ethnicity
    , pt.admissionheight
    , pt.admissionweight
    , pt.dischargeweight
    , ROUND(pt.unitdischargeoffset/60) AS icu_los_hours
  FROM patient pt
  LEFT JOIN hospital h
    ON pt.hospitalid = h.hospitalid
  LEFT JOIN apachepatientresult ap
    ON pt.patientunitstayid = ap.patientunitstayid
    AND ap.apacheversion = 'IV'
  ORDER BY pt.uniquepid, pt.unitvisitnumber, pt.age
);
```

### 3. Extract Vital Signs (Pivot Format)

```sql
DROP TABLE IF EXISTS pivoted_vital CASCADE;
CREATE TABLE pivoted_vital AS
WITH nc AS (
  SELECT
      patientunitstayid
    , nursingchartoffset
    , nursingchartentryoffset
    , CASE
        WHEN nursingchartcelltypevallabel = 'Heart Rate'
         AND nursingchartcelltypevalname = 'Heart Rate'
         AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
         AND nursingchartvalue NOT IN ('-','.')
            THEN CAST(nursingchartvalue AS NUMERIC)
        ELSE NULL END
      AS heartrate
    , CASE
        WHEN nursingchartcelltypevallabel = 'Non-Invasive BP'
         AND nursingchartcelltypevalname = 'Non-Invasive BP Systolic'
         AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
         AND nursingchartvalue NOT IN ('-','.')
            THEN CAST(nursingchartvalue AS NUMERIC)
        ELSE NULL END
      AS nibp_systolic
    -- similar handling for other vital signs...
  FROM nursecharting
  WHERE nursingchartcelltypecat IN ('Vital Signs', 'Scores', 'Other Vital Signs and Infusions')
)
SELECT
    patientunitstayid
  , nursingchartoffset AS chartoffset
  , nursingchartentryoffset AS entryoffset
  , AVG(CASE WHEN heartrate BETWEEN 25 AND 225 THEN heartrate ELSE NULL END) AS heartrate
  , AVG(CASE WHEN nibp_systolic BETWEEN 25 AND 250 THEN nibp_systolic ELSE NULL END) AS nibp_systolic
  -- ... other vital signs
FROM nc
WHERE heartrate IS NOT NULL
   OR nibp_systolic IS NOT NULL
   -- ...
GROUP BY patientunitstayid, nursingchartoffset, nursingchartentryoffset
ORDER BY patientunitstayid, nursingchartoffset, nursingchartentryoffset;
```

### 4. Extract Laboratory Tests (Pivot Format)

```sql
DROP TABLE IF EXISTS pivoted_lab CASCADE;
CREATE TABLE pivoted_lab AS
WITH vw0 AS (
  SELECT
      patientunitstayid
    , labname
    , labresultoffset
    , labresultrevisedoffset
  FROM lab
  WHERE labname IN (
      'albumin', 'total bilirubin', 'BUN', 'calcium', 'chloride',
      'creatinine', 'bedside glucose', 'glucose', 'bicarbonate', 'Total CO2',
      'Hct', 'Hgb', 'PT - INR', 'PTT', 'lactate',
      'platelets x 1000', 'potassium', 'sodium', 'WBC x 1000', '-bands',
      'ALT (SGPT)', 'AST (SGOT)', 'alkaline phos.'
    )
  GROUP BY patientunitstayid, labname, labresultoffset, labresultrevisedoffset
  HAVING COUNT(DISTINCT labresult) <= 1
),
-- Get the last revised result
vw1 AS (
  SELECT
      lab.patientunitstayid
    , lab.labname
    , lab.labresultoffset
    , lab.labresultrevisedoffset
    , lab.labresult
    , ROW_NUMBER() OVER (
        PARTITION BY lab.patientunitstayid, lab.labname, lab.labresultoffset
        ORDER BY lab.labresultrevisedoffset DESC
      ) AS rn
  FROM lab
  INNER JOIN vw0
    ON  lab.patientunitstayid = vw0.patientunitstayid
    AND lab.labname = vw0.labname
    AND lab.labresultoffset = vw0.labresultoffset
    AND lab.labresultrevisedoffset = vw0.labresultrevisedoffset
  WHERE
      (lab.labname = 'albumin' AND lab.labresult >= 0.5 AND lab.labresult <= 6.5)
    OR (lab.labname = 'creatinine' AND lab.labresult >= 0.1 AND lab.labresult <= 28.28)
    -- ... plausibility checks for other tests
)
SELECT
    patientunitstayid
  , labresultoffset AS chartoffset
  , MAX(CASE WHEN labname = 'albumin' THEN labresult ELSE NULL END) AS albumin
  , MAX(CASE WHEN labname = 'total bilirubin' THEN labresult ELSE NULL END) AS bilirubin
  , MAX(CASE WHEN labname = 'BUN' THEN labresult ELSE NULL END) AS bun
  , MAX(CASE WHEN labname = 'creatinine' THEN labresult ELSE NULL END) AS creatinine
  -- ... other tests
FROM vw1
WHERE rn = 1
GROUP BY patientunitstayid, labresultoffset
ORDER BY patientunitstayid, labresultoffset;
```

---

## Python Code Examples

### Using psycopg2 to Query eICU Data

```python
import psycopg2
import pandas as pd

# Database connection parameters
conn_params = {
    'host': 'localhost',
    'port': 5432,
    'database': 'eicu',
    'user': 'your_username',
    'password': 'your_password'
}

# Query vital signs data
def get_vital_signs(patient_id, hour_window=24):
    """
    Extract vital signs data for a specified patient in the first N hours after ICU admission.
    
    Args:
        patient_id: ICU stay ID (patientunitstayid)
        hour_window: Time window in hours, default 24 hours
    """
    query = """
        SELECT 
            vp.patientunitstayid,
            vp.observationoffset,
            vp.temperature,
            vp.sao2,
            vp.heartrate,
            vp.respiration,
            vp.systemicsystolic,
            vp.systemicdiastolic,
            vp.systemicmean
        FROM vitalperiodic vp
        WHERE vp.patientunitstayid = %(patient_id)s
          AND vp.observationoffset >= 0 
          AND vp.observationoffset < %(max_offset)s
        ORDER BY vp.observationoffset;
    """
    
    # Convert hours to minutes
    max_offset = hour_window * 60
    
    with psycopg2.connect(**conn_params) as conn:
        df = pd.read_sql(
            query, 
            conn,
            params={'patient_id': patient_id, 'max_offset': max_offset}
        )
    
    return df

# Query laboratory test data
def get_lab_results(patient_id, hour_window=24):
    """
    Extract laboratory test data for a specified patient in the first N hours after ICU admission.
    """
    query = """
        SELECT 
            l.patientunitstayid,
            l.labresultoffset,
            l.labname,
            l.labresult
        FROM lab l
        WHERE l.patientunitstayid = %(patient_id)s
          AND l.labresultoffset >= 0 
          AND l.labresultoffset < %(max_offset)s
          AND l.labname IN (
              'creatinine', 'BUN', 'sodium', 'potassium', 
              'chloride', 'bicarbonate', 'calcium'
          )
          AND l.labresult IS NOT NULL
        ORDER BY l.labresultoffset;
    """
    
    max_offset = hour_window * 60
    
    with psycopg2.connect(**conn_params) as conn:
        df = pd.read_sql(
            query,
            conn,
            params={'patient_id': patient_id, 'max_offset': max_offset}
        )
    
    return df

# Usage example
if __name__ == '__main__':
    # Replace with actual patientunitstayid
    patient_id = 123456
    
    # Get vital signs
    vitals_df = get_vital_signs(patient_id, hour_window=24)
    print(f"Got {len(vitals_df)} vital signs records")
    print(vitals_df.head())
    
    # Get laboratory tests
    labs_df = get_lab_results(patient_id, hour_window=24)
    print(f"\nGot {len(labs_df)} lab results records")
    print(labs_df.head())
```

---

## Reference Links

- **Official SchemaSpy Documentation**: https://lcp.mit.edu/eicu-schema-spy/index.html
- **eICU Data Introduction**: https://eicu-crd.mit.edu/
- **Access Application**: https://physionet.org/content/eicu-crd/2.0/
- **eICU-code Official Repository**: https://github.com/MIT-LCP/eicu-code
- **MIMIC-code Official Repository**: https://github.com/MIT-LCP/mimic-code

---

**Last Updated**: 2026-05-20  
**Updated By**: Wukong (thoroughly revised based on official SchemaSpy documentation and MIT-LCP/eicu-code repository)
