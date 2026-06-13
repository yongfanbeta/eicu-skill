# eICU 2.0 数据库 Schema 参考

> **⚠️ 重要说明（2026-05-20 修订）**: 本文档根据 **MIT 官方 SchemaSpy** 和 **eICU-code 仓库** 进行修订。
>
> **关键事实**：
> - ✅ `vitalperiodic` 和 `vitalaperiodic` 表**都存在**（这是 eICU 2.0 的真实表）
> - ✅ 生命体征数据来自 **多个表**：`vitalperiodic`（周期性）、`vitalaperiodic`（非周期性）、`nursecharting`（护理记录）
> - ✅ 所有表名和字段名均为 **小写**，但使用 **驼色命名**（如 `heartrate` 不是 `heart_rate`）
>
> **数据来源**: 
> - [MIT eICU SchemaSpy 官方文档](https://lcp.mit.edu/eicu-schema-spy/index.html)
> - [eICU-code 官方仓库](https://github.com/MIT-LCP/eicu-code)
> 
> **数据库**: eicu.eicu_crd  
> **总表数**: 31 张表  
> **总记录数**: 约 4.57 亿条

---

## 目录

1. [⚠️ 重要更正与最佳实践](#重要更正与最佳实践)
2. [核心表详细说明](#核心表详细说明)
   - [patient](#patient) ⭐ 最重要
   - [lab](#lab) ⭐ 实验室检查
   - [nursecharting](#nursecharting) ⭐ 生命体征来源
   - [hospital](#hospital)
   - [apachepatientresult](#apachepatientresult)
   - [diagnosis](#diagnosis)
   - [admissiondx](#admissiondx)
   - [medication](#medication)
   - [infusiondrug](#infusiondrug)
   - [admissiondrug](#admissiondrug)
   - [treatment](#treatment)
   - [intakeoutput](#intakeoutput)
   - [respiratorycare](#respiratorycare)
   - [respiratorycharting](#respiratorycharting)
   - [nursecare](#nursecare)
   - [nurseassessment](#nurseassessment)
   - [pasthistory](#pasthistory)
   - [microlab](#microlab)
   - [physicalexam](#physicalexam)
   - [note](#note)
   - [careplangeneral](#careplangeneral)
   - [careplangoal](#careplangoal)
   - [careplancareprovider](#careplancareprovider)
   - [careplaninfectiousdisease](#careplaninfectiousdisease)
   - [careplaneol](#careplaneol)
   - [customlab](#customlab)
   - [allergy](#allergy)
   - [apacheapsvar](#apacheapsvar)
   - [apachepredvar](#apachepredvar)
3. [表间关系](#表间关系)
4. [重要说明](#重要说明)

---

## ⚠️ 重要更正与最佳实践

### 1. 生命体征数据来源（关键！）

**✅ 正确认知**：eICU **确实有** `vitalperiodic` 和 `vitalaperiodic` 表！

**数据来源**：生命体征数据分布在 **三个表**：
1. `vitalperiodic` - 周期性生命体征（每 1-5 分钟，1.46 亿条记录）
2. `vitalaperiodic` - 非周期性生命体征（有创监测，2507 万条记录）
3. `nursecharting` - 护理记录中的生命体征（1.51 亿条记录）

**查询建议**：
- 优先使用 `vitalperiodic` 和 `vitalaperiodic`（结构化程度高）
- 补充使用 `nursecharting`（更详细但需解析）

```sql
SELECT 
    patientunitstayid
  , nursingchartoffset       -- 记录偏移时间（分钟）
  , nursingchartentryoffset  -- 录入偏移时间（分钟）
  , nursingchartcelltypecat  -- 大类（如 'Vital Signs'）
  , nursingchartcelltypevallabel  -- 中类（如 'Heart Rate', 'Non-Invasive BP'）
  , nursingchartcelltypevalname   -- 小类（如 'Heart Rate', 'Non-Invasive BP Systolic'）
  , nursingchartvalue        -- 值（文本或数字）
FROM nursecharting
WHERE nursingchartcelltypecat IN ('Vital Signs', 'Scores', 'Other Vital Signs and Infusions');
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
GROUP BY patientunitstayid, nursingchartoffset
ORDER BY patientunitstayid, nursingchartoffset;
```

### 2. 实验室检查数据

**✅ 正确方法**：使用 `lab` 表，`labname` 直接是检查名称（**不需要 itemid 映射**）

```sql
SELECT 
    patientunitstayid
  , labresultoffset       -- 结果偏移时间（分钟）
  , labresultrevisedoffset  -- 结果修订偏移时间（分钟）
  , labname               -- 检查名称（字符串）
  , labresult             -- 检查结果（数值）
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
  -- 添加合理性检查
  AND (
        (labname = 'albumin' AND labresult >= 0.5 AND labresult <= 6.5)
    OR (labname = 'creatinine' AND labresult >= 0.1 AND labresult <= 28.28)
    -- ... 其他检查的范围检查
  );
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

### 3. 诊断数据

**✅ 正确方法**：使用 `patient` 表的 `apacheadmissiondx` 字段（APACHE IV 入院诊断）

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
- `'ACS'`（急性冠脉综合征）
- `'ChestPainUnknown'`（不明原因胸痛）
- `'CHF'`（充血性心力衰竭）
- `'CardiacArrest'`（心脏骤停）
- `'CABG'`（冠脉搭桥）
- `'ValveDz'`（心脏瓣膜病）
- `'PNA'`（肺炎）
- `'RespMedOther'`（呼吸系统其他疾病）
- `'Asthma-Emphys'`（哮喘/肺气肿）
- `'GIBleed'`（消化道出血）
- `'GIObstruction'`（肠梗阻）
- `'CVA'`（脑卒中）
- `'Neuro'`（神经系统疾病）
- `'Coma'`（昏迷）
- `'Overdose'`（药物过量）
- `'Sepsis'`（脓毒症）
- `'ARF'`（急性肾衰竭）
- `'DKA'`（糖尿病酮症酸中毒）
- `'Trauma'`（创伤）
- `'Other'`（其他）

### 4. 时间表示（重要！）

eICU 使用 **偏移量（分钟）** 表示时间，所有偏移量相对于 **ICU 入住时间**：

- `offset = 0`: ICU 入住时刻
- `offset = 60`: 入住后 60 分钟（1 小时）
- `offset = 1440`: 入住后 1440 分钟（24 小时）
- **负值**: 表示入住前的记录（如有）

**关键时间字段**：
- `hospitaladmitoffset` - 医院入院偏移量
- `hospitaldischargeoffset` - 医院出院偏移量
- `unitdischargeoffset` - ICU 出院偏移量（**用于计算 ICU LOS**）
- `nursingchartoffset` - 护理记录偏移时间
- `labresultoffset` - 实验室结果偏移时间
- `diagnosisoffset` - 诊断偏移时间

**计算 ICU 住院时长（小时）**：
```sql
ROUND(unitdischargeoffset/60) AS icu_los_hours
```

---

## 核心表详细说明

### patient

患者基本信息表，每次 ICU 住院一行。

**主键**: patientunitstayid  
**记录数**: 200,859

| 字段 | 类型 | 说明 |
|------|------|------|
| patientunitstayid | int4 | ICU 住院 ID (PK) |
| uniquepid | varchar | 患者唯一标识 |
| patienthealthsystemstayid | int4 | 医院住院标识 |
| hospitalid | int4 | 医院 ID |
| unitvisitnumber | int4 | ICU 住院次数 |
| unittype | varchar | ICU 类型 (Med-Surg/Neuro/CCU/etc) |
| apacheadmissiondx | varchar | APACHE IV 入院诊断（重要！） |
| age | varchar | 年龄（文本，>89 为 "91"）|
| gender | varchar | 性别 ('Male'/'Female') |
| ethnicity | varchar | 种族 |
| hospitaladmitoffset | int4 | 医院入院偏移量（分钟）|
| hospitaldischargeoffset | int4 | 医院出院偏移量（分钟）|
| unitdischargeoffset | int4 | ICU 出院偏移量（分钟）|
| hospitaldischargestatus | varchar | 医院出院状态 ('Alive'/'Expired') |
| admissionheight | numeric | 入院身高 (cm) |
| admissionweight | numeric | 入院体重 (kg) |
| dischargeweight | numeric | 出院体重 (kg) |

**重要字段说明**:
- `hospitaldischargestatus = 'Expired'` 表示患者死亡
- `unitdischargeoffset` 用于计算 ICU 住院时长（除以 60 得到小时）
- `apacheadmissiondx` 是 APACHE IV 入院诊断，用于疾病分组
- `age` 字段是文本类型，>89 岁的患者显示为 "91"（数据截断处理）

**计算 ICU 住院时长（小时）**：
```sql
ROUND(unitdischargeoffset/60) AS icu_los_hours
```

---

### lab

实验室检查结果。

**主键**: labid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 39,132,531

| 字段 | 类型 | 说明 |
|------|------|------|
| labid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| labresultoffset | int4 | 结果偏移量（分钟，相对于 ICU 入住）|
| labresultrevisedoffset | int4 | 修订结果偏移量（分钟）|
| labname | varchar(256) | 检查名称（如 "creatinine", "sodium"）|
| labresult | numeric(11,4) | 结果值（数值）|
| labresulttext | varchar(255) | 结果文本（用于定性结果）|

**常用 labname 值**:
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

**注意**:
- `labresult` 为数值结果，`labresulttext` 为文本结果（如 "positive", "negative"）
- `labresultrevisedoffset` 可用于识别修正后的结果
- 查询时务必添加合理性检查（如 `labresult >= 0.1 AND labresult <= 28.28` 对于肌酐）

---

### nursecharting

护理图表记录（**生命体征数据的来源！**）。

**主键**: nursechartid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 151,604,232

| 字段 | 类型 | 说明 |
|------|------|------|
| nursechartid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| nursingchartoffset | int4 | 记录偏移时间（分钟，相对于 ICU 入住）|
| nursingchartentryoffset | int4 | 录入偏移时间（分钟）|
| nursingchartcelltypecat | varchar | 大类（如 'Vital Signs'）|
| nursingchartcelltypevallabel | varchar | 中类（如 'Heart Rate', 'Non-Invasive BP'）|
| nursingchartcelltypevalname | varchar | 小类（如 'Heart Rate', 'Non-Invasive BP Systolic'）|
| nursingchartvalue | varchar | 值（文本或数字）|

**常用 nursingchartcelltypevallabel 值**（生命体征）：
- `'Heart Rate'` - 心率
- `'Respiratory Rate'` - 呼吸频率
- `'O2 Saturation'` - 血氧饱和度
- `'Non-Invasive BP'` - 无创血压（包含 Systolic/Diastolic/Mean）
- `'Temperature'` - 体温
- `'Invasive BP'` - 有创血压（包含 Systolic/Diastolic/Mean）
- `'MAP (mmHg)'` - 平均动脉压
- `'Arterial Line MAP (mmHg)'` - 动脉线平均动脉压

**提取生命体征的常用模式**（从 eicu-code/pivoted/pivoted-vital.sql 学习）：

```sql
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
  , CASE
      WHEN nursingchartcelltypevallabel = 'Non-Invasive BP'
       AND nursingchartcelltypevalname = 'Non-Invasive BP Diastolic'
       AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
       AND nursingchartvalue NOT IN ('-','.')
          THEN CAST(nursingchartvalue AS NUMERIC)
      ELSE NULL END
    AS nibp_diastolic
  , CASE
      WHEN nursingchartcelltypevallabel = 'Non-Invasive BP'
       AND nursingchartcelltypevalname = 'Non-Invasive BP Mean'
       AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
       AND nursingchartvalue NOT IN ('-','.')
          THEN CAST(nursingchartvalue AS NUMERIC)
      ELSE NULL END
    AS nibp_mean
  , CASE
      WHEN nursingchartcelltypevallabel = 'Temperature'
       AND nursingchartcelltypevalname = 'Temperature (C)'
       AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
       AND nursingchartvalue NOT IN ('-','.')
          THEN CAST(nursingchartvalue AS NUMERIC)
      ELSE NULL END
    AS temperature
  , CASE
      WHEN nursingchartcelltypevallabel = 'Temperature'
       AND nursingchartcelltypevalname = 'Temperature Location'
          THEN nursingchartvalue
      ELSE NULL END
    AS temperaturelocation
  -- 类似处理其他生命体征...
FROM nursecharting
WHERE nursingchartcelltypecat IN ('Vital Signs', 'Scores', 'Other Vital Signs and Infusions')
ORDER BY patientunitstayid, nursingchartoffset;
```

**注意**:
- 这是最大的表（1.51 亿条记录），查询时务必使用 `patientunitstayid` 过滤
- `nursingchartvalue` 可能是文本（如 '80s' 表示 '80多岁'），需要清洗
- 使用正则表达式 `~ '^[-]?[0-9]+[.]?[0-9]*$'` 检查是否为数字
- 查询时添加合理性检查（如 `heartrate >= 25 AND heartrate <= 225`）

---

### hospital

医院信息表。

**主键**: hospitalid  
**记录数**: 208

| 字段 | 类型 | 说明 |
|------|------|------|
| hospitalid | int4 | 医院 ID (PK) |
| region | varchar | 医院所在地区 |

---

### apachepatientresult

APACHE IV 评分结果。

**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 297,064

| 字段 | 类型 | 说明 |
|------|------|------|
| patientunitstayid | int4 | 患者 ID (FK) |
| apacheversion | varchar | APACHE 版本（应为 "IV"）|
| apachescore | numeric | APACHE IV 评分 |
| predictedicumortality | numeric | 预测 ICU 死亡率 |
| predictedhospitalmortality | numeric | 预测住院死亡率 |
| actualicumortality | int4 | 实际 ICU 死亡 (1/0) |
| actualhospitalmortality | int4 | 实际住院死亡 (1/0) |
| actualiculos | numeric | 实际 ICU 住院时间（小时）|
| actualhospitallos | numeric | 实际住院时间（小时）|

**注意**:
- `predictedicumortality` 是 APACHE IV 模型预测的死亡概率
- `actualicumortality = 1` 表示 ICU 期间死亡

---

### diagnosis

诊断信息。

**主键**: diagnosisid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 2,710,672

| 字段 | 类型 | 说明 |
|------|------|------|
| diagnosisid | int4 | 诊断 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| diagnosisoffset | int4 | 诊断偏移量（分钟）|
| diagnosisstring | varchar(200) | 诊断名称 |
| icd9code | varchar(100) | ICD-9 编码 |
| diagnosispriority | varchar(10) | 诊断优先级（'Primary'/'Secondary'）|

**注意**:
- `diagnosispriority = 'Primary'` 表示主要诊断
- 一个患者可以有多个诊断记录
- eicu-code 主要使用 `patient.apacheadmissiondx` 字段，而不是这个表

---

### admissiondx

入院诊断。

**主键**: admissiondxid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 626,858

| 字段 | 类型 | 说明 |
|------|------|------|
| admissiondxid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| admissiondxoffset | int4 | 诊断偏移量（分钟）|
| admissiondxstring | varchar | 入院诊断名称 |
| icd9code | varchar | ICD-9 编码 |

---

### medication

药物处方记录。

**主键**: medicationid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 7,301,853

| 字段 | 类型 | 说明 |
|------|------|------|
| medicationid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| drugorderoffset | int4 | 医嘱偏移量（分钟）|
| drugstartoffset | int4 | 开始给药偏移量（分钟）|
| drugname | varchar(220) | 药物名称 |
| dosage | varchar(60) | 剂量 |
| routeadmin | varchar(120) | 给药途径 |
| frequency | varchar(255) | 给药频率 |
| drugstopoffset | int4 | 停止给药偏移量（分钟）|

**注意**:
- `routeadmin` 常见值: "IV", "PO", "IM", "SC", etc.
- `frequency` 表示给药频率（如 "q6h" 表示每 6 小时一次）

---

### infusiondrug

输注药物记录（详细记录输注速率等）。

**主键**: infusiondrugid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 4,803,719

| 字段 | 类型 | 说明 |
|------|------|------|
| infusiondrugid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| infusionoffset | int4 | 输注偏移量（分钟）|
| drugname | varchar(255) | 药物名称 |
| drugrate | varchar(255) | 输注速率 |
| drugamount | varchar(255) | 药物总量 |
| volumeoffluid | varchar(255) | 液体容量 |

**注意**:
- `drugrate` 通常表示输注速度（如 "10 mL/hr"）
- `drugamount` 表示总药量
- 此表与 `medication` 表可能包含重叠数据，但 `infusiondrug` 更详细

---

### admissiondrug

入院时药物。

**主键**: admissiondrugid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 874,920

| 字段 | 类型 | 说明 |
|------|------|------|
| admissiondrugid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| admissiondrugoffset | int4 | 药物偏移量（分钟）|
| admissiondrugname | varchar | 药物名称 |

---

### treatment

治疗措施记录。

**主键**: treatmentid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 3,688,745

| 字段 | 类型 | 说明 |
|------|------|------|
| treatmentid | int4 | 治疗 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| treatmentoffset | int4 | 治疗开始偏移量（分钟）|
| treatmentendoffset | int4 | 治疗结束偏移量（分钟）|
| treatmentstring | varchar | 治疗名称/描述 |

**常用治疗**:
- 机械通气: `'Mechanical Ventilation'`
- 血液净化: `'CRRT'` / `'Hemodialysis'`
- 血管活性药物: `'Vasopressors'`

---

### intakeoutput

出入量记录。

**主键**: intakeoutputid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 12,030,289

| 字段 | 类型 | 说明 |
|------|------|------|
| intakeoutputid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| intakeoutputoffset | int4 | 记录偏移量（分钟）|
| intakeoutputstring | varchar | 出入量描述 |
| celllabel | varchar | 分类标签 |
| cellattribute | varchar | 属性 |
| cellvaluenumeric | numeric | 数值 |
| cellvalueuom | varchar | 单位 |

---

### respiratorycare

呼吸治疗记录。

**主键**: respiratorycareid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 865,381

| 字段 | 类型 | 说明 |
|------|------|------|
| respiratorycareid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| respiratorycareoffset | int4 | 治疗偏移量（分钟）|
| respiratorycarestring | varchar | 呼吸治疗描述 |

---

### respiratorycharting

呼吸治疗图表记录。

**主键**: respiratorychartid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 20,168,176

| 字段 | 类型 | 说明 |
|------|------|------|
| respiratorychartid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| respiratorychartoffset | int4 | 记录偏移量（分钟）|
| respiratorychartstring | varchar | 呼吸记录描述 |
| celllabel | varchar | 分类标签 |
| cellattribute | varchar | 属性 |
| cellvaluetext | varchar | 文本值 |
| cellvaluenumeric | numeric | 数值 |

---

### nursecare

护理操作记录。

**主键**: nursecareid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 8,311,132

| 字段 | 类型 | 说明 |
|------|------|------|
| nursecareid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| nursecareoffset | int4 | 操作偏移量（分钟）|
| nursecarestring | varchar | 护理操作描述 |

---

### nurseassessment

护理评估记录。

**主键**: nurseassessid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 15,602,498

| 字段 | 类型 | 说明 |
|------|------|------|
| nurseassessid | int4 | 评估 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| nurseassessoffset | int4 | 评估偏移量（分钟）|
| nurseassessstring | varchar | 评估内容/描述 |

---

### pasthistory

既往史记录。

**主键**: pasthistoryid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 1,149,180

| 字段 | 类型 | 说明 |
|------|------|------|
| pasthistoryid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| pasthistoryoffset | int4 | 记录偏移量（分钟）|
| pasthistorystring | varchar | 既往史描述 |
| icd9code | varchar | ICD-9 编码（如有）|

---

### microlab

微生物培养结果。

**主键**: microlabid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 16,996

| 字段 | 类型 | 说明 |
|------|------|------|
| microlabid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| microlaboffset | int4 | 培养偏移量（分钟）|
| culturesite | varchar | 培养部位 |
| organism | varchar | 微生物名称 |
| antibiotic | varchar | 抗生素 |
| susceptibility | varchar | 药敏结果 |

---

### physicalexam

体格检查记录。

**主键**: physicalexamid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 9,212,316

| 字段 | 类型 | 说明 |
|------|------|------|
| physicalexamid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| physicalexamoffset | int4 | 检查偏移时间（分钟）|
| physicalexamstring | varchar | 体格检查描述 |

---

### note

临床笔记/文书记录。

**主键**: noteid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 2,254,179

| 字段 | 类型 | 说明 |
|------|------|------|
| noteid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| noteoffset | int4 | 笔记偏移量（分钟）|
| notetype | varchar | 笔记类型 |
| notetext | text | 笔记内容 |

---

### careplangeneral

通用护理计划。

**主键**: careplangeneralid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 3,115,018

| 字段 | 类型 | 说明 |
|------|------|------|
| careplangeneralid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| careplangeneraloffset | int4 | 计划偏移量（分钟）|
| careplangeneralstring | varchar | 护理计划描述 |

---

### careplangoal

护理目标。

**主键**: careplangoalid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 504,139

| 字段 | 类型 | 说明 |
|------|------|------|
| careplangoalid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| careplangoaloffset | int4 | 目标偏移量（分钟）|
| careplangoalstring | varchar | 护理目标描述 |

---

### careplancareprovider

护理提供者。

**主键**: careplancareproviderid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 502,765

| 字段 | 类型 | 说明 |
|------|------|------|
| careplancareproviderid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| careplancareprovideroffset | int4 | 提供者偏移量（分钟）|
| careplancareproviderstring | varchar | 护理提供者描述 |

---

### careplaninfectiousdisease

感染性疾病护理计划。

**主键**: careplaninfectiousdiseaseid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 8,056

| 字段 | 类型 | 说明 |
|------|------|------|
| careplaninfectiousdiseaseid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| careplaninfectiousdiseaseoffset | int4 | 计划偏移量（分钟）|
| careplaninfectiousdiseasestring | varchar | 感染性疾病护理计划描述 |

---

### careplaneol

临终护理计划。

**主键**: careplaneolid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 1,433

| 字段 | 类型 | 说明 |
|------|------|------|
| careplaneolid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| careplaneoloffset | int4 | 计划偏移量（分钟）|
| careplaneolstring | varchar | 临终护理计划描述 |

---

### customlab

自定义实验室检查。

**主键**: customlabid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 1,082

| 字段 | 类型 | 说明 |
|------|------|------|
| customlabid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| customlaboffset | int4 | 自定义实验室偏移量（分钟）|
| customlabstring | varchar | 自定义实验室检查描述 |

---

### allergy

过敏信息。

**主键**: allergyid  
**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 251,949

| 字段 | 类型 | 说明 |
|------|------|------|
| allergyid | int4 | 记录 ID (PK) |
| patientunitstayid | int4 | 患者 ID (FK) |
| allergyoffset | int4 | 过敏偏移量（分钟）|
| allergystring | varchar | 过敏描述 |

---

### apacheapsvar

APACHE APS 变量。

**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 171,177

| 字段 | 类型 | 说明 |
|------|------|------|
| patientunitstayid | int4 | 患者 ID (FK) |
| apsvarid | int4 | APS 变量 ID |
| apsvarvalue | numeric | APS 变量值 |

---

### apachepredvar

APACHE 预测变量。

**外键**: patientunitstayid → patient.patientunitstayid  
**记录数**: 171,177

| 字段 | 类型 | 说明 |
|------|------|------|
| patientunitstayid | int4 | 患者 ID (FK) |
| predvarid | int4 | 预测变量 ID |
| predvarvalue | numeric | 预测变量值 |

---

## 表间关系

```
patient (patientunitstayid) [PK]
  |
  +-- lab (1:N)
  +-- nursecharting (1:N)  ⭐ 生命体征来源
  +-- apachepatientresult (1:1)
  +-- diagnosis (1:N)
  +-- admissiondx (1:N)
  +-- medication (1:N)
  +-- infusiondrug (1:N)
  +-- admissiondrug (1:N)
  +-- treatment (1:N)
  +-- intakeoutput (1:N)
  +-- respiratorycare (1:N)
  +-- respiratorycharting (1:N)
  +-- nursecare (1:N)
  +-- nurseassessment (1:N)
  +-- pasthistory (1:N)
  +-- microlab (1:N)
  +-- physicalexam (1:N)
  +-- note (1:N)
  +-- careplangeneral (1:N)
  +-- careplangoal (1:N)
  +-- careplancareprovider (1:N)
  +-- careplaninfectiousdisease (1:N)
  +-- careplaneol (1:N)
  +-- customlab (1:N)
  +-- allergy (1:N)
  +-- apacheapsvar (1:1)
  +-- apachepredvar (1:1)
```

**关系说明**:
- 所有表通过 `patientunitstayid` 与 `patient` 表关联
- 一对多关系（1:N）：一个患者可以有多个生命体征、实验室、诊断等记录
- 一对一关系（1:1）：如 `apachepatientresult`，每个患者只有一条 APACHE 评分结果

---

## 重要说明

### 1. 时间表示

eICU 使用 **偏移量（分钟）** 表示时间，所有偏移量相对于 **ICU 入住时间**：

- `offset = 0`: ICU 入住时刻
- `offset = 60`: 入住后 60 分钟（1 小时）
- `offset = 1440`: 入住后 1440 分钟（24 小时）
- **负值**: 表示入住前的记录（如有）

**优势**: 便于计算时间差，不受实际时间戳影响。

### 2. 患者标识

- **patientunitstayid**: ICU 住院 ID（每次 ICU 入住唯一）
- **uniquepid**: 唯一患者 ID（跨多次住院标识同一患者）
- **patienthealthsystemstayid**: 健康系统住院 ID（每次住院唯一）

**注意**: eICU 中不使用 `stay_id`（这是 MIMIC-IV 的字段）。

### 3. 字段命名规范

- 所有字段名使用 **小写**
- 单词间无空格或下划线（如 `patientunitstayid`）
- 偏移量字段统一为 `*offset`（如 `nursingchartoffset`, `labresultoffset`）

### 4. 数据类型

- 数值型: `int4`, `int8`, `numeric(precision, scale)`, `float8`
- 字符型: `varchar(length)`, `text`
- 日期时间: 无独立日期时间字段，统一使用偏移量

### 5. 数据质量与性能优化

**数据质量**:
- **缺失值多**: eICU 的数据完整性低于 MIMIC-IV
- **异常值**: 查询时务必加合理性检查（如 `heartrate >= 25 AND heartrate <= 225`）
- **文本值**: 部分生命体征值是文本（如 '80s' 表示 '80多岁'），需要清洗

**性能优化**:

**大表**（根据 eicu-code 统计）：
- `nursecharting`: 约 1.51 亿条记录
- `lab`: 约 3913 万条记录
- `patient`: 约 20 万条记录
- `intakeoutput`: 约 1203 万条记录
- `respiratorycharting`: 约 2016 万条记录

**优化建议**：
1. **使用 `patientunitstayid` 过滤**（最重要！）
2. **限定时间范围**（`nursingchartoffset BETWEEN 0 AND 1440`）
3. **添加 `LIMIT` 子句**（测试时）
4. **使用 `EXPLAIN` 分析查询计划**
5. **考虑创建物化视图**（`CREATE MATERIALIZED VIEW`）

---

## 参考链接

- **eICU-code 官方仓库**: https://github.com/MIT-LCP/eicu-code
- **eICU 数据介绍**: https://eicu-crd.mit.edu/
- **访问申请**: https://physionet.org/content/eicu-crd/2.0/
- **MIMIC-code 官方仓库**: https://github.com/MIT-LCP/mimic-code

---

**最后更新**: 2026-05-20  
**更新人**: 悟空（基于 MIT-LCP/eicu-code 官方仓库彻底修订）
