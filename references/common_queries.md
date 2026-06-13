# eICU 2.0 常用查询模板

## 目录

1. [患者基本信息](#1-患者基本信息)
2. [APACHE IV 评分](#2-apache-iv-评分)
3. [机械通气信息](#3-机械通气信息)
4. [血管活性药物](#4-血管活性药物)
5. [既往史](#5-既往史)
6. [护理评估](#6-护理评估)
7. [微生物培养](#7-微生物培养)
8. [出入量](#8-出入量)
9. [护理记录](#9-护理记录)

---

## 1. 患者基本信息

```sql
SELECT
    p.patientunitstayid,
    p.age,
    p.gender,
    p.ethnicity,
    p.admissionheight,
    p.admissionweight,
    p.dischargeweight,
    p.apacheadmissiondx,
    p.unittype,
    p.unitstaytype,
    p.unitdischargestatus,
    p.hospitaldischargestatus,
    p.unitdischargeoffset,
    p.hospitaldischargeoffset,
    -- APACHE 结果
    a.predictedicumortality,
    a.predictedhospitalmortality,
    a.actualicumortality,
    a.actualhospitalmortality,
    a.actualiculos,
    a.actualhospitallos
FROM patient p
LEFT JOIN apachepatientresult a
    ON p.patientunitstayid = a.patientunitstayid
ORDER BY p.patientunitstayid;
```

**重要字段说明**:
- `unitdischargestatus`: ICU 出院状态（`'Alive'` / `'Expired'`）
- `hospitaldischargestatus`: 住院出院状态（`'Alive'` / `'Expired'`）
- `unitdischargeoffset`: ICU 住院时长（分钟）
- `hospitaldischargeoffset`: 住院时长（分钟）

---

## 2. APACHE IV 评分

### 2.1 APACHE IV 预测结果

```sql
SELECT
    p.patientunitstayid,
    p.age,
    p.gender,
    a.apacheversion,
    a.predictedicumortality,
    a.predictedhospitalmortality,
    a.actualicumortality,
    a.actualhospitalmortality,
    a.actualiculos,
    a.actualhospitallos,
    a.actualventdays,
    a.actualventday1
FROM patient p
INNER JOIN apachepatientresult a
    ON p.patientunitstayid = a.patientunitstayid
ORDER BY p.patientunitstayid;
```

### 2.2 APACHE APS 变量（生理参数）

```sql
SELECT
    aps.patientunitstayid,
    aps.apacheapsvarid,
    aps.apacheapsvaroffset,
    aps.apacheapsvarname,
    aps.apacheapsvarvalue
FROM apacheapsvar aps
WHERE aps.patientunitstayid = %(patient_id)s
ORDER BY aps.apacheapsvaroffset, aps.apacheapsvarid;
```

### 2.3 APACHE 预测变量

```sql
SELECT
    apv.patientunitstayid,
    apv.apachepredvarid,
    apv.apachepredvarname,
    apv.apachepredvarvalue
FROM apachepredvar apv
WHERE apv.patientunitstayid = %(patient_id)s
ORDER BY apv.apachepredvarid;
```

---

## 3. 机械通气信息

eICU 的机械通气信息分布在多个表中。

### 3.1 通过 treatment 表判断是否通气

```sql
SELECT
    t.patientunitstayid,
    t.treatmentoffset,
    t.treatmentendoffset,
    t.treatmentstring
FROM treatment t
WHERE t.treatmentstring ILIKE '%ventilat%'
   OR t.treatmentstring ILIKE '%mechanical%'
ORDER BY t.patientunitstayid, t.treatmentoffset;
```

### 3.2 通过 apachepatientresult 判断第一天通气

```sql
SELECT
    p.patientunitstayid,
    a.actualventday1 AS ventilated_day1,
    a.actualventdays AS total_vent_days
FROM patient p
INNER JOIN apachepatientresult a
    ON p.patientunitstayid = a.patientunitstayid
WHERE a.actualventday1 = 1
ORDER BY p.patientunitstayid;
```

### 3.3 通过 respiratorycare 获取呼吸治疗信息

```sql
SELECT
    rc.patientunitstayid,
    rc.respiratorycareoffset,
    rc.respiratorycarestring
FROM respiratorycare rc
WHERE rc.patientunitstayid = %(patient_id)s
  AND rc.respiratorycareoffset BETWEEN 0 AND 1440
ORDER BY rc.respiratorycareoffset;
```

### 3.4 通过 respiratorycharting 获取详细呼吸记录

```sql
SELECT
    rch.patientunitstayid,
    rch.respiratorychartoffset,
    rch.respiratorychartstring,
    rch.celllabel,
    rch.cellattribute,
    rch.cellvaluetext,
    rch.cellvaluenumeric
FROM respiratorycharting rch
WHERE rch.patientunitstayid = %(patient_id)s
  AND rch.respiratorychartoffset BETWEEN 0 AND 1440
ORDER BY rch.respiratorychartoffset;
```

**注意**: eICU **没有** `fio2`, `peep`, `peakpressure`, `plateaupressure` 等字段（这些在 MIMIC-IV 的 `ventilation` 表中）。eICU 的呼吸参数记录在 `respiratorycharting` 表的 `celllabel`/`cellvaluetext` 中。

---

## 4. 血管活性药物

### 4.1 infusiondrug 表（输注药物，更精确）

```sql
SELECT
    id.patientunitstayid,
    id.infusionoffset,
    id.drugname,
    id.drugratel,
    id.infusionrate,
    id.drugamount,
    id.volumeoffluid,
    id.patientweight
FROM infusiondrug id
WHERE id.patientunitstayid = %(patient_id)s
  AND id.drugname ILIKE ANY(ARRAY[
      '%norepinephrine%', '%levarterenol%', '%levophed%',
      '%epinephrine%', '%adrenaline%',
      '%vasopressin%', '%pitressin%',
      '%dopamine%',
      '%dobutamine%',
      '%phenylephrine%', '%neosynephrine%',
      '%milrinone%'
  ])
  AND id.infusionoffset BETWEEN 0 AND 1440
ORDER BY id.infusionoffset;
```

### 4.2 medication 表（药物处方）

```sql
SELECT
    m.patientunitstayid,
    m.drugorderoffset,
    m.drugstartoffset,
    m.drugstopoffset,
    m.drugname,
    m.dosage,
    m.routeadmin,
    m.frequency,
    m.infusionrate,
    m.prn
FROM medication m
WHERE m.patientunitstayid = %(patient_id)s
  AND m.drugname ILIKE ANY(ARRAY[
      '%norepinephrine%', '%epinephrine%', '%vasopressin%',
      '%dopamine%', '%dobutamine%', '%phenylephrine%', '%milrinone%'
  ])
  AND m.drugstartoffset BETWEEN 0 AND 1440
ORDER BY m.drugstartoffset;
```

---

## 5. 既往史

### 5.1 查看所有既往史描述

```sql
-- 查看所有不重复的既往史描述（前 100 个）
SELECT DISTINCT pasthistorystring, COUNT(*) AS cnt
FROM pasthistory
GROUP BY pasthistorystring
ORDER BY cnt DESC
LIMIT 100;
```

### 5.2 查询特定患者的既往史

```sql
SELECT
    ph.patientunitstayid,
    ph.pasthistoryoffset,
    ph.pasthistorystring,
    ph.icd9code
FROM pasthistory ph
WHERE ph.patientunitstayid = %(patient_id)s
ORDER BY ph.pasthistoryoffset;
```

### 5.3 统计各类合并症患者数

```sql
SELECT
    'diabetes' AS comorbidity,
    COUNT(DISTINCT ph.patientunitstayid) AS num_patients
FROM pasthistory ph
WHERE ph.pasthistorystring ILIKE '%diabetes%'

UNION ALL

SELECT
    'copd' AS comorbidity,
    COUNT(DISTINCT ph.patientunitstayid) AS num_patients
FROM pasthistory ph
WHERE ph.pasthistorystring ILIKE '%copd%'
   OR ph.pasthistorystring ILIKE '%chronic obstructive%'

UNION ALL

SELECT
    'renal_dialysis' AS comorbidity,
    COUNT(DISTINCT ph.patientunitstayid) AS num_patients
FROM pasthistory ph
WHERE ph.pasthistorystring ILIKE '%dialysis%'
   OR ph.pasthistorystring ILIKE '%renal%'

UNION ALL

SELECT
    'heart_failure' AS comorbidity,
    COUNT(DISTINCT ph.patientunitstayid) AS num_patients
FROM pasthistory ph
WHERE ph.pasthistorystring ILIKE '%heart failure%'
   OR ph.pasthistorystring ILIKE '%chf%'

ORDER BY num_patients DESC;
```

---

## 6. 护理评估

### 6.1 疼痛评估

```sql
SELECT
    na.patientunitstayid,
    na.nurseassessoffset,
    na.nurseassessstring
FROM nurseassessment na
WHERE na.patientunitstayid = %(patient_id)s
  AND na.nurseassessoffset BETWEEN 0 AND 1440
  AND na.nurseassessstring ILIKE '%pain%'
ORDER BY na.nurseassessoffset;
```

### 6.2 护理操作记录

```sql
SELECT
    nc.patientunitstayid,
    nc.nursecareoffset,
    nc.nursecarestring
FROM nursecare nc
WHERE nc.patientunitstayid = %(patient_id)s
  AND nc.nursecareoffset BETWEEN 0 AND 1440
ORDER BY nc.nursecareoffset
LIMIT 100;
```

---

## 7. 微生物培养

### 7.1 提取微生物培养结果

```sql
SELECT
    ml.patientunitstayid,
    ml.microlaboffset,
    ml.culturesite,
    ml.organism,
    ml.antibiotic,
    ml.susceptibility
FROM microlab ml
WHERE ml.patientunitstayid = %(patient_id)s
  AND ml.microlaboffset BETWEEN 0 AND 1440
  AND ml.organism IS NOT NULL 
  AND ml.organism != ''
ORDER BY ml.microlaboffset;
```

### 7.2 统计常见病原体

```sql
SELECT
    ml.organism,
    COUNT(DISTINCT ml.patientunitstayid) AS num_patients,
    COUNT(*) AS num_cultures
FROM microlab ml
WHERE ml.organism IS NOT NULL 
  AND ml.organism != ''
GROUP BY ml.organism
ORDER BY num_patients DESC
LIMIT 30;
```

---

## 8. 出入量

### 8.1 提取出入量记录

```sql
SELECT
    io.patientunitstayid,
    io.intakeoutputoffset,
    io.intakeoutputstring,
    io.celllabel,
    io.cellattribute,
    io.cellvaluenumeric,
    io.cellvalueuom
FROM intakeoutput io
WHERE io.patientunitstayid = %(patient_id)s
  AND io.intakeoutputoffset BETWEEN 0 AND 1440
ORDER BY io.intakeoutputoffset;
```

### 8.2 计算第一天总入量和总出量

```sql
SELECT
    io.patientunitstayid,
    SUM(CASE WHEN io.celllabel ILIKE '%intake%' 
              OR io.celllabel ILIKE '%input%' 
              OR io.celllabel ILIKE '%in%'
             THEN io.cellvaluenumeric ELSE 0 END) AS total_intake,
    SUM(CASE WHEN io.celllabel ILIKE '%output%' 
              OR io.celllabel ILIKE '%urine%' 
              OR io.celllabel ILIKE '%out%'
             THEN io.cellvaluenumeric ELSE 0 END) AS total_output
FROM intakeoutput io
WHERE io.intakeoutputoffset BETWEEN 0 AND 1440
  AND io.cellvaluenumeric IS NOT NULL
GROUP BY io.patientunitstayid
ORDER BY io.patientunitstayid;
```

---

## 9. 护理记录

### 9.1 提取护理记录

```sql
SELECT
    nch.patientunitstayid,
    nch.nursechartoffset,
    nch.nursechartstring,
    nch.celllabel,
    nch.cellattribute,
    nch.cellvaluetext,
    nch.cellvaluenumeric
FROM nursecharting nch
WHERE nch.patientunitstayid = %(patient_id)s
  AND nch.nursechartoffset BETWEEN 0 AND 1440
ORDER BY nch.nursechartoffset
LIMIT 1000;
```

**注意**: `nursecharting` 表有 **151,476,983 条记录** (~1.51 亿)，查询时务必：
- 使用 `patientunitstayid` 过滤
- 限定 `nursechartoffset` 范围
- 添加 `LIMIT` 子句

### 9.2 提取特定护理记录（如格拉斯哥昏迷评分 GCS）

```sql
SELECT
    nch.patientunitstayid,
    nch.nursechartoffset,
    nch.celllabel,
    nch.cellvaluetext,
    nch.cellvaluenumeric
FROM nursecharting nch
WHERE nch.patientunitstayid = %(patient_id)s
  AND nch.nursechartoffset BETWEEN 0 AND 1440
  AND (
    nch.celllabel ILIKE '%gcs%'
    OR nch.celllabel ILIKE '%glasgow%'
    OR nch.nursechartstring ILIKE '%gcs%'
  )
ORDER BY nch.nursechartoffset;
```

---

## 注意事项

### 1. 字段名规范

✅ **正确** (小写):
```sql
SELECT p.patientunitstayid, p.age, p.gender
FROM patient p
WHERE p.unitdischargestatus = 'Expired'
```

❌ **错误** (camelCase):
```sql
SELECT p.patientUnitStayID, p.age, p.sex  -- 字段不存在！
FROM patient p
WHERE p.dischargeStatus = 'Expired'
```

### 2. 表名规范

✅ **正确** (小写):
```sql
SELECT * FROM vitalperiodic;
SELECT * FROM vitalaperiodic;
SELECT * FROM apacheapsvar;
SELECT * FROM microlab;
```

❌ **错误** (错误表名):
```sql
SELECT * FROM vitalSignDerived;  -- 表不存在！
SELECT * FROM apachePatientVars;  -- 表不存在！
SELECT * FROM microbiology;  -- 表名错误，应为 microlab
```

### 3. 时间表示

- 所有时间偏移量字段（如 `observationoffset`, `labresultoffset`, `diagnosisoffset`）单位为 **分钟**
- 相对于 **ICU 入住时间**:
  - `offset = 0`: ICU 入住时刻
  - `offset = 1440`: 入住后 24 小时
- 负值表示入住前的记录（如有）

### 4. 数据质量

- **缺失值多**: eICU 的数据完整性低于 MIMIC-IV
- **异常值**: 查询时务必加合理性检查
- **模糊匹配**: 使用 `ILIKE` (不区分大小写) 进行文本匹配

### 5. 性能优化

- **大表** (`vitalperiodic`, `nursecharting`, `lab`):
  - 使用 `patientunitstayid` 过滤
  - 限定时间范围 (`observationoffset BETWEEN 0 AND 1440`)
  - 添加 `LIMIT` 子句（测试时）
  - 使用 `EXPLAIN` 分析查询计划

---

## 参考链接

- **官方 SchemaSpy 文档**: https://lcp.mit.edu/eicu-schema-spy/index.html
- **eICU 数据介绍**: https://eicu-crd.mit.edu/
- **访问申请**: https://physionet.org/content/eicu-crd/2.0/

---

**最后更新**: 2026-05-20  
**更新人**: 悟空（基于 MIT 官方 SchemaSpy 文档修正）
