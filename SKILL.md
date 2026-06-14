---
name: eicu-skill
description: Query the eICU Collaborative Research Database 2.0. Generates SQL and Python code for extracting ICU data including vital signs, labs, GCS, vasopressors, blood gas, oxygenation, and diagnoses from PostgreSQL. Supports both direct SQL queries and Python (psycopg2) scripts.
---

# eICU 2.0 数据提取技能

> **⚠️ 重要更正（2026-05-20）**: 本文档已根据 **官方 SchemaSpy 文档** 和 **MIT-LCP/eicu-code 仓库** 进行彻底修订。
>
> **关键修正**：
> - ✅ `vitalperiodic` 和 `vitalaperiodic` 表**都存在**（我之前错误地认为它们不存在）
> - ✅ 字段名是 **驼色命名**（`heartrate` 不是 `heart_rate` 也不是 `heartRate`）
> - ✅ 生命体征数据来自 **多个表**（`vitalperiodic`, `vitalaperiodic`, `nursecharting`）

从 eICU Collaborative Research Database 2.0 中提取重症监护数据。提供 SQL 模板和 Python 代码，用户自行处理数据库连接。

## 连接方式

用户提供自己的 PostgreSQL 连接参数。本技能只提供查询代码。

## 支持的查询类型

| 查询类型 | 参考文件 |
|---------|----------|
| 患者基本信息 | [references/schema.md](references/schema.md) (patient 表) |
| 生命体征（vitalperiodic） | [references/vital_signs.md](references/vital_signs.md) |
| 生命体征（vitalaperiodic） | [references/vital_signs.md](references/vital_signs.md) |
| 生命体征（nursecharting） | [references/vital_signs_nursecharting.md](references/vital_signs_nursecharting.md) |
| 体格检查（GCS 等） | [references/gcs.md](references/gcs.md) |
| 实验室检查 | [references/labs.md](references/labs.md) |
| 血气分析 | [references/blood_gas.md](references/blood_gas.md) |
| 尿量（Urine Output） | [references/urine_output.md](references/urine_output.md) |
| 升压药（Vasopressors） | [references/vasopressors.md](references/vasopressors.md) |
| 输液药物（Infusions） | [references/infusions.md](references/infusions.md) |
| 体重（Weight） | [references/weight.md](references/weight.md) |
| 氧合（Oxygenation） | [references/oxygenation.md](references/oxygenation.md) |
| 诊断与 APACHE 分组 | [references/diagnoses.md](references/diagnoses.md) |
| 药物（Medications） | [references/medications.md](references/medications.md) |
| 数据库 Schema | [references/schema.md](references/schema.md) |
| 常用查询模板 | [references/common_queries.md](references/common_queries.md) |

## 工作流程

1. 确认用户需要的查询类型
2. 读取对应的 references 文件获取 SQL/Python 模板
3. 根据用户具体需求调整查询条件
4. 同时提供 SQL 和 Python 两种实现

## eICU 与 MIMIC-IV 的关键区别

| 特征 | eICU 2.0 | MIMIC-IV |
|------|----------|----------|
| **时间表示** | 偏移量（分钟） | 绝对时间戳 |
| **生命体征来源** | `vitalperiodic` + `vitalaperiodic` + `nursecharting` | `chartevents` 表（通过 `d_items` 映射） |
| **实验室检查** | `lab` 表（`labname` 直接是名称） | `labevents` 表（通过 `d_labitems` 映射） |
| **患者ID** | `patientunitstayid` | `stay_id` |
| **数据量** | 约 20 万患者 | 约 5 万患者 |
| **字段名** | 驼色命名（如 `heartrate`） | 全小写，snake_case |

## 核心表结构（正确）

### 1. 患者基本信息 - `patient` 表

```sql
SELECT 
    patientunitstayid  -- ICU 住院唯一标识
  , uniquepid           -- 患者唯一标识
  , patienthealthsystemstayid  -- 医院住院标识
  , hospitalid          -- 医院ID
  , unitvisitnumber     -- ICU 住院次数
  , unittype            -- ICU 类型
  , apacheadmissiondx   -- APACHE IV 入院诊断（重要！）
  , age                 -- 年龄（截断处理：>89 岁显示为 91）
  , gender              -- 性别（'Male'/'Female'）
  , ethnicity           -- 种族
  , hospitaladmitoffset  -- 医院入院偏移量（分钟）
  , hospitaldischargeoffset  -- 医院出院偏移量（分钟）
  , unitdischargeoffset  -- ICU 出院偏移量（分钟）
  , hospitaldischargestatus  -- 医院出院状态（'Alive'/'Expired'）
  , admissionheight     -- 入院身高（cm）
  , admissionweight     -- 入院体重（kg）
  , dischargeweight     -- 出院体重（kg）
FROM patient;
```

**计算 ICU 住院时长（小时）**：
```sql
ROUND(unitdischargeoffset/60) AS icu_los_hours
```

### 2. 周期性生命体征 - `vitalperiodic` 表

**表名**: `vitalperiodic`  
**记录数**: 146,671,642 条  
**说明**: 周期性生命体征测量，通常每 1-5 分钟记录一次。

```sql
SELECT 
    vitalperiodicid     -- 记录 ID (PK)
  , patientunitstayid  -- 患者 ID (FK)
  , observationoffset   -- 观察偏移量（分钟，相对于 ICU 入住）
  , temperature         -- 体温 (°C)
  , sao2               -- 血氧饱和度 (%)
  , heartrate          -- 心率 (bpm)
  , respiration        -- 呼吸频率 (breaths/min)
  , cvp                -- 中心静脉压 (mmHg)
  , etco2              -- 呼气末 CO2 (mmHg)
  , systemicsystolic   -- 有创收缩压 (mmHg)
  , systemicdiastolic  -- 有创舒张压 (mmHg)
  , systemicmean       -- 有创平均动脉压 (mmHg)
  , pasystolic         -- 肺动脉收缩压 (mmHg)
  , padiastolic        -- 肺动脉舒张压 (mmHg)
  , pamean             -- 肺动脉平均压 (mmHg)
  , st1                -- ST 段导联 1 (mV)
  , st2                -- ST 段导联 2 (mV)
  , st3                -- ST 段导联 3 (mV)
  , icp                -- 颅内压 (mmHg)
FROM vitalperiodic
WHERE observationoffset >= 0 AND observationoffset < 1440  -- 入住后第一个 24 小时
  AND patientunitstayid = %(patient_id)s
ORDER BY observationoffset;
```

**注意**:
- 所有血压值均为有创测量（动脉导管）
- `observationoffset` 是相对于 ICU 入住时间的分钟数
- 添加合理性检查：`heartrate BETWEEN 25 AND 225`

---

### 3. 非周期性生命体征 - `vitalaperiodic` 表

**表名**: `vitalaperiodic`  
**记录数**: 25,075,074 条  
**说明**: 非周期性生命体征测量（通常是有创监测数据）。

```sql
SELECT 
    vitalaperiodicid   -- 记录 ID (PK)
  , patientunitstayid   -- 患者 ID (FK)
  , observationoffset    -- 观察偏移量（分钟）
  , noninvasivesystolic  -- 无创收缩压 (mmHg)
  , noninvasivediastolic -- 无创舒张压 (mmHg)
  , noninvasivemean     -- 无创平均动脉压 (mmHg)
  , paop               -- 肺动脉闭塞压 (mmHg)
  , cardiacoutput       -- 心输出量 (L/min)
  , cardiacinput        -- 心脏指数 (L/min/m²)
  , svr                -- 全身血管阻力 (dyn·s·cm⁻⁵)
  , svri               -- 全身血管阻力指数 (dyn·s·cm⁻⁵/m²)
  , pvr                -- 肺血管阻力 (dyn·s·cm⁻⁵)
  , pvri               -- 肺血管阻力指数 (dyn·s·cm⁻⁵/m²)
FROM vitalaperiodic
WHERE observationoffset >= 0 AND observationoffset < 1440  -- 入住后第一个 24 小时
  AND patientunitstayid = %(patient_id)s
ORDER BY observationoffset;
```

**注意**:
- 此表主要包含有创血流动力学监测数据
- `paop` (肺动脉闭塞压) 也称 PCWP (肺毛细血管楔压)
- `svr`/`svri` 和 `pvr`/`pvri` 用于评估血管阻力

---

### 4. 护理记录中的生命体征 - `nursecharting` 表

**表名**: `nursecharting`  
**记录数**: 151,604,232 条  
**说明**: 护理图表记录（最详细的护理数据，包含生命体征、评分等）。

```sql
SELECT 
    nursechartid              -- 记录 ID (PK)
  , patientunitstayid        -- 患者 ID (FK)
  , nursingchartoffset       -- 记录偏移时间（分钟）
  , nursingchartentryoffset  -- 录入偏移时间（分钟）
  , nursingchartcelltypecat  -- 大类（如 'Vital Signs'）
  , nursingchartcelltypevallabel  -- 中类（如 'Heart Rate', 'Non-Invasive BP'）
  , nursingchartcelltypevalname   -- 小类（如 'Heart Rate', 'Non-Invasive BP Systolic'）
  , nursingchartvalue        -- 值（文本或数字）
FROM nursecharting
WHERE nursingchartcelltypecat IN ('Vital Signs', 'Scores', 'Other Vital Signs and Infusions')
  AND patientunitstayid = %(patient_id)s
  AND nursingchartoffset >= 0 AND nursingchartoffset < 1440  -- 入住后第一个 24 小时
ORDER BY nursingchartoffset;
```

**提取生命体征的常用模式**（从 eicu-code 学习）：

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
               -- 类似处理其他生命体征...
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

### 5. 实验室检查 - `lab` 表

**表名**: `lab`  
**记录数**: 39,132,531 条  
**说明**: 实验室检查结果。`labname` 直接是检查名称（**不需要 itemid 映射**）。

```sql
SELECT 
    labid                   -- 记录 ID (PK)
  , patientunitstayid       -- 患者 ID (FK)
  , labresultoffset         -- 结果偏移时间（分钟）
  , labresultrevisedoffset  -- 结果修订偏移时间（分钟）
  , labname                 -- 检查名称（字符串）
  , labresult               -- 检查结果（数值）
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
  AND labresultoffset >= 0 AND labresultoffset < 1440  -- 入住后第一个 24 小时
  -- 添加合理性检查
  AND (
        (labname = 'albumin' AND labresult >= 0.5 AND labresult <= 6.5)
    OR (labname = 'creatinine' AND labresult >= 0.1 AND labresult <= 28.28)
    -- ... 其他检查的范围检查
  )
ORDER BY labresultoffset;
```

**常用 labname 值**：
- 白蛋白: `'albumin'`
- 总胆红素: `'total bilirubin'`
- 血尿素氮: `'BUN'`
- 钙: `'calcium'`
- 氯化物: `'chloride'`
- 肌酐: `'creatinine'`
- 血糖: `'bedside glucose'` 或 `'glucose'`
- 碳酸氢盐: `'bicarbonate'` 或 `'Total CO2'`
- 红细胞压积: `'Hct'`
- 血红蛋白: `'Hgb'`
- 国际标准化比值: `'PT - INR'`
- 部分凝血活酶时间: `'PTT'`
- 乳酸: `'lactate'`
- 血小板: `'platelets x 1000'`
- 钾: `'potassium'`
- 钠: `'sodium'`
- 白细胞: `'WBC x 1000'`
-  bands: `'-bands'`
- 谷丙转氨酶: `'ALT (SGPT)'`
- 谷草转氨酶: `'AST (SGOT)'`
- 碱性磷酸酶: `'alkaline phos.'`

---

### 6. 诊断 - `patient` 表（APACHE 诊断）

eICU 的诊断信息主要来自 `patient` 表的 `apacheadmissiondx` 字段（APACHE IV 入院诊断）。

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

**APACHE 诊断分组**（从 eicu-code 学习）：

eicu-code 提供了 `diagnosis/apache-groups.sql`，将 `apacheadmissiondx` 分组为：
- `'ACS'` (急性冠脉综合征)
- `'ChestPainUnknown'` (不明原因胸痛)
- `'CHF'` (充血性心力衰竭)
- `'CardiacArrest'` (心脏骤停)
- `'CABG'` (冠脉搭桥)
- `'ValveDz'` (心脏瓣膜病)
- `'PNA'` (肺炎)
- `'RespMedOther'` (呼吸系统其他疾病)
- `'Asthma-Emphys'` (哮喘/肺气肿)
- `'GIBleed'` (消化道出血)
- `'GIObstruction'` (肠梗阻)
- `'CVA'` (脑卒中)
- `'Neuro'` (神经系统疾病)
- `'Coma'` (昏迷)
- `'Overdose'` (药物过量)
- `'Sepsis'` (脓毒症)
- `'ARF'` (急性肾衰竭)
- `'DKA'` (糖尿病酮症酸中毒)
- `'Trauma'` (创伤)
- `'Other'` (其他)

---

## 时间表示（重要！）

eICU 使用 **偏移量（分钟）** 表示时间，所有偏移量相对于 **ICU 入住时间**：

- `offset = 0`: ICU 入住时刻
- `offset = 60`: 入住后 60 分钟（1 小时）
- `offset = 1440`: 入住后 1440 分钟（24 小时）
- **负值**: 表示入住前的记录（如有）

**关键时间字段**：
- `hospitaladmitoffset` - 医院入院偏移量
- `hospitaldischargeoffset` - 医院出院偏移量
- `unitdischargeoffset` - ICU 出院偏移量（**用于计算 ICU LOS**）
- `observationoffset` - 生命体征记录偏移时间
- `nursingchartoffset` - 护理记录偏移时间
- `labresultoffset` - 实验室结果偏移时间
- `diagnosisoffset` - 诊断偏移时间

**计算 ICU 住院时长（小时）**：
```sql
ROUND(unitdischargeoffset/60) AS icu_los_hours
```

---

## 数据质量与性能优化

### 数据质量

- **缺失值多**: eICU 的数据完整性低于 MIMIC-IV
- **异常值**: 查询时务必加合理性检查（如 `heartrate BETWEEN 25 AND 225`）
- **文本值**: 部分生命体征值是文本（如 '80s' 表示 '80多岁'），需要清洗

### 性能优化

**大表**（根据 eicu-code 统计）：
- `nursecharting`: 约 1.51 亿条记录
- `lab`: 约 3913 万条记录
- `vitalperiodic`: 约 1.46 亿条记录
- `vitalaperiodic`: 约 2507 万条记录
- `patient`: 约 20 万条记录

**优化建议**：
1. **使用 `patientunitstayid` 过滤**（最重要！）
2. **限定时间范围**（`observationoffset BETWEEN 0 AND 1440`）
3. **添加 `LIMIT` 子句**（测试时）
4. **使用 `EXPLAIN` 分析查询计划**
5. **考虑创建物化视图**（`CREATE MATERIALIZED VIEW`）

---

## 常用查询场景

### 1. 提取患者基本信息

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

### 2. 提取 ICU 住院详细信息（物化视图）

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

### 3. 提取生命体征（Pivot 格式）

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
    -- 类似处理其他生命体征...
  FROM nursecharting
  WHERE nursingchartcelltypecat IN ('Vital Signs', 'Scores', 'Other Vital Signs and Infusions')
)
SELECT
    patientunitstayid
  , nursingchartoffset AS chartoffset
  , nursingchartentryoffset AS entryoffset
  , AVG(CASE WHEN heartrate BETWEEN 25 AND 225 THEN heartrate ELSE NULL END) AS heartrate
  , AVG(CASE WHEN nibp_systolic BETWEEN 25 AND 250 THEN nibp_systolic ELSE NULL END) AS nibp_systolic
  -- ... 其他生命体征
FROM nc
WHERE heartrate IS NOT NULL
   OR nibp_systolic IS NOT NULL
   -- ...
GROUP BY patientunitstayid, nursingchartoffset, nursingchartentryoffset
ORDER BY patientunitstayid, nursingchartoffset, nursingchartentryoffset;
```

### 4. 提取实验室检查（Pivot 格式）

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
-- 获取最后一次修订的结果
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
    -- ... 其他检查的范围检查
)
SELECT
    patientunitstayid
  , labresultoffset AS chartoffset
  , MAX(CASE WHEN labname = 'albumin' THEN labresult ELSE NULL END) AS albumin
  , MAX(CASE WHEN labname = 'total bilirubin' THEN labresult ELSE NULL END) AS bilirubin
  , MAX(CASE WHEN labname = 'BUN' THEN labresult ELSE NULL END) AS bun
  , MAX(CASE WHEN labname = 'creatinine' THEN labresult ELSE NULL END) AS creatinine
  -- ... 其他检查
FROM vw1
WHERE rn = 1
GROUP BY patientunitstayid, labresultoffset
ORDER BY patientunitstayid, labresultoffset;
```

---

## Python 代码示例

### 使用 psycopg2 查询 eICU 数据

```python
import psycopg2
import pandas as pd

# 数据库连接参数
conn_params = {
    'host': 'localhost',
    'port': 5432,
    'database': 'eicu',
    'user': 'your_username',
    'password': 'your_password'
}

# 查询生命体征数据
def get_vital_signs(patient_id, hour_window=24):
    """
    提取指定患者在 ICU 入住后前 N 小时的生命体征数据
    
    Args:
        patient_id: ICU 住院 ID (patientunitstayid)
        hour_window: 时间窗口（小时），默认 24 小时
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
    
    # 转换小时为分钟
    max_offset = hour_window * 60
    
    with psycopg2.connect(**conn_params) as conn:
        df = pd.read_sql(
            query, 
            conn,
            params={'patient_id': patient_id, 'max_offset': max_offset}
        )
    
    return df

# 查询实验室检查数据
def get_lab_results(patient_id, hour_window=24):
    """
    提取指定患者在 ICU 入住后前 N 小时的实验室检查数据
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

# 使用示例
if __name__ == '__main__':
    # 替换为实际的 patientunitstayid
    patient_id = 123456
    
    # 获取生命体征
    vitals_df = get_vital_signs(patient_id, hour_window=24)
    print(f"Got {len(vitals_df)} vital signs records")
    print(vitals_df.head())
    
    # 获取实验室检查
    labs_df = get_lab_results(patient_id, hour_window=24)
    print(f"\nGot {len(labs_df)} lab results records")
    print(labs_df.head())
```

---

## 参考链接

- **官方 SchemaSpy 文档**: https://lcp.mit.edu/eicu-schema-spy/index.html
- **eICU 数据介绍**: https://eicu-crd.mit.edu/
- **访问申请**: https://physionet.org/content/eicu-crd/2.0/
- **eICU-code 官方仓库**: https://github.com/MIT-LCP/eicu-code
- **MIMIC-code 官方仓库**: https://github.com/MIT-LCP/mimic-code

---

**最后更新**: 2026-05-20  
**更新人**: 悟空（基于官方 SchemaSpy 文档和 MIT-LCP/eicu-code 仓库彻底修订）
