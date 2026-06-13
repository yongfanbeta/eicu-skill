# eICU 2.0 生命体征查询（使用 nursecharting 表）

**参考**: 
- eicu-code 官方实现: `D:\Project\eicu-code\concepts\pivoted\pivoted-vital.sql`
- eICU 2.0 SchemaSpy 文档: https://lcp.mit.edu/eicu-schema-spy/

---

## 目录

1. [为什么使用 nursecharting 表？](#为什么使用-nursecharting-表)
2. [指标映射](#指标映射)
3. [SQL 模板](#sql-模板)
4. [Python 模板](#python-模板)
5. [注意事项](#注意事项)

---

## 为什么使用 nursecharting 表？

eicu-code **官方实现使用 `nursecharting` 表**提取生命体征，而非 `vitalperiodic` 或 `vitalaperiodic`。

**原因**:
1. **数据完整性更好**: `nursecharting` 包含护士记录的所有生命体征（包括手动输入的）
2. **标准化更高**: eicu-code 已经处理了数据清洗和异常值检查
3. **性能更好**: 可以通过 `nursingchartcelltypecat` 快速过滤

**两种方法对比**:

| 特性 | 方法 1（`vitalperiodic` + `vitalaperiodic`） | 方法 2（`nursecharting`） |
|------|-----------------------------------------------|-----------------------------|
| **数据来源** | 床旁监护仪自动采集 | 护士手动记录 + 监护仪数据 |
| **数据频率** | 高（每 1-5 分钟） | 中等（按护理记录频率） |
| **数据完整性** | 可能缺失（监护仪故障） | 更好（人工补充） |
| **异常值检查** | 需要手动添加 | eicu-code 已包含 |
| **推荐使用场景** | 需要高频数据（如心率变异性） | 需要可靠的生命体征（如计算 SOFA 评分） |

**建议**: 
- 如果需要**高频数据**（如心率变异性分析），使用方法 1
- 如果需要**可靠的生命体征**（如计算 SOFA 评分），使用方法 2（本文件）

---

## 指标映射

`nursecharting` 表使用 `nursingchartcelltypevlabel` 和 `nursingchartcelltypevalname` 来标识不同类型的生命体征。

**重要**: 提取时务必使用**精确匹配**（`nursingchartcelltypevlabel = 'xxx'` 和 `nursingchartcelltypevalname = 'xxx'`）。

### 1. "主要" 生命体征（eicu-code 放入 `pivoted_vital` 表）

| 指标 | nursingchartcelltypevlabel | nursingchartcelltypevalname | 异常值检查 | 说明 |
|------|---------------------------|----------------------------|--------------|------|
| 心率 | `'Heart Rate'` | `'Heart Rate'` | `>= 25 AND <= 225` | bpm |
| 呼吸频率 | `'Respiratory Rate'` | `'Respiratory Rate'` | `>= 0 AND <= 60` | breaths/min |
| 血氧饱和度 | `'O2 Saturation'` | `'O2 Saturation'` | `>= 0 AND <= 100` | % |
| 无创收缩压 | `'Non-Invasive BP'` | `'Non-Invasive BP Systolic'` | `>= 25 AND <= 250` | mmHg |
| 无创舒张压 | `'Non-Invasive BP'` | `'Non-Invasive BP Diastolic'` | `>= 1 AND <= 200` | mmHg |
| 无创平均动脉压 | `'Non-Invasive BP'` | `'Non-Invasive BP Mean'` | `>= 1 AND <= 250` | mmHg |
| 体温 | `'Temperature'` | `'Temperature (C)'` | `>= 25 AND <= 46` | °C |
| 体温测量部位 | `'Temperature'` | `'Temperature Location'` | 文本字段 | 如 "Oral", "Rectal" |
| 有创收缩压 | `'Invasive BP'` | `'Invasive BP Systolic'` | `>= 1 AND <= 300` | mmHg |
| 有创舒张压 | `'Invasive BP'` | `'Invasive BP Diastolic'` | `>= 1 AND <= 200` | mmHg |
| 有创平均动脉压 | `'Invasive BP'` | `'Invasive BP Mean'` | `>= 1 AND <= 250` | mmHg |
| 平均动脉压（有创，另一种来源） | `'MAP (mmHg)'` | `'Value'` | `>= 1 AND <= 250` | mmHg |
| 平均动脉压（有创，动脉置管） | `'Arterial Line MAP (mmHg)'` | `'Value'` | `>= 1 AND <= 250` | mmHg |

### 2. "次要" 生命体征（eicu-code 放入 `pivoted_vital_other` 表）

| 指标 | nursingchartcelltypevlabel | nursingchartcelltypevalname | 异常值检查 | 说明 |
|------|---------------------------|----------------------------|--------------|------|
| 肺动脉收缩压 | `'PA'` | `'PA Systolic'` | `>= 0 AND <= 1000` | mmHg |
| 肺动脉舒张压 | `'PA'` | `'PA Diastolic'` | `>= 0 AND <= 1000` | mmHg |
| 肺动脉平均压 | `'PA'` | `'PA Mean'` | `>= 0 AND <= 1000` | mmHg |
| 每搏输出量 | `'SV'` | `'SV'` | `>= 0 AND <= 1000` | mL |
| 心输出量 | `'CO'` | `'CO'` | `>= 0 AND <= 1000` | L/min |
| 全身血管阻力 | `'SVR'` | `'SVR'` | `>= 0 AND <= 1000` | dyn·s·cm⁻⁵ |
| 颅内压 | `'ICP'` | `'ICP'` | `>= 0 AND <= 1000` | mmHg |
| 心脏指数 | `'CI'` | `'CI'` | `>= 0 AND <= 1000` | L/min/m² |
| 全身血管阻力指数 | `'SVRI'` | `'SVRI'` | `>= 0 AND <= 1000` | dyn·s·cm⁻⁵/m² |
| 脑灌注压 | `'CPP'` | `'CPP'` | `>= 0 AND <= 1000` | mmHg |
| 混合静脉血氧饱和度 | `'SVO2'` | `'SVO2'` | `>= 0 AND <= 1000` | % |
| 肺动脉闭塞压 | `'PAOP'` | `'PAOP'` | `>= 0 AND <= 1000` | mmHg |
| 肺血管阻力 | `'PVR'` | `'PVR'` | `>= 0 AND <= 1000` | dyn·s·cm⁻⁵ |
| 肺血管阻力指数 | `'PVRI'` | `'PVRI'` | `>= 0 AND <= 1000` | dyn·s·cm⁻⁵/m² |
| 腹内压 | `'IAP'` | `'IAP'` | `>= 0 AND <= 1000` | mmHg |

⚠️ **注意**: 以上 `nursingchartcelltypevlabel` 和 `nursingchartcelltypevalname` 值来自 eicu-code 官方实现。**不同站点可能略有不同**，建议先查询确认：

```sql
-- 查看所有不重复的 nursingchartcelltypevlabel 和 nursingchartcelltypevalname（前 100 个）
SELECT DISTINCT nursingchartcelltypevlabel, nursingchartcelltypevalname, COUNT(*) AS cnt
FROM nursecharting
WHERE nursingchartcelltypecat IN ('Vital Signs', 'Other Vital Signs and Infusions')
GROUP BY nursingchartcelltypevlabel, nursingchartcelltypevalname
ORDER BY cnt DESC
LIMIT 100;
```

---

## SQL 模板

### 1. 提取 ICU 第一天的生命体征（长格式，基于 eicu-code）

```sql
-- 基于 eicu-code 官方实现：D:\Project\eicu-code\concepts\pivoted\pivoted-vital.sql
-- 提取 ICU 第一天（0-1440 分钟）的生命体征（长格式）
WITH nc AS (
    SELECT
        patientunitstayid,
        nursingchartoffset,
        nursingchartentryoffset,
        CASE
            WHEN nursingchartcelltypevlabel = 'Heart Rate'
             AND nursingchartcelltypevalname = 'Heart Rate'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS heartrate,
        CASE
            WHEN nursingchartcelltypevlabel = 'Respiratory Rate'
             AND nursingchartcelltypevalname = 'Respiratory Rate'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS respiratoryrate,
        CASE
            WHEN nursingchartcelltypevlabel = 'O2 Saturation'
             AND nursingchartcelltypevalname = 'O2 Saturation'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS o2saturation,
        CASE
            WHEN nursingchartcelltypevlabel = 'Non-Invasive BP'
             AND nursingchartcelltypevalname = 'Non-Invasive BP Systolic'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS nibp_systolic,
        CASE
            WHEN nursingchartcelltypevlabel = 'Non-Invasive BP'
             AND nursingchartcelltypevalname = 'Non-Invasive BP Diastolic'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS nibp_diastolic,
        CASE
            WHEN nursingchartcelltypevlabel = 'Non-Invasive BP'
             AND nursingchartcelltypevalname = 'Non-Invasive BP Mean'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS nibp_mean,
        CASE
            WHEN nursingchartcelltypevlabel = 'Temperature'
             AND nursingchartcelltypevalname = 'Temperature (C)'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS temperature,
        CASE
            WHEN nursingchartcelltypevlabel = 'Temperature'
             AND nursingchartcelltypevalname = 'Temperature Location'
                THEN nursingchartvalue
            ELSE NULL
        END AS temperaturelocation,
        CASE
            WHEN nursingchartcelltypevlabel = 'Invasive BP'
             AND nursingchartcelltypevalname = 'Invasive BP Systolic'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS ibp_systolic,
        CASE
            WHEN nursingchartcelltypevlabel = 'Invasive BP'
             AND nursingchartcelltypevalname = 'Invasive BP Diastolic'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS ibp_diastolic,
        CASE
            WHEN nursingchartcelltypevlabel = 'Invasive BP'
             AND nursingchartcelltypevalname = 'Invasive BP Mean'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            -- other map fields
            WHEN nursingchartcelltypevlabel = 'MAP (mmHg)'
             AND nursingchartcelltypevalname = 'Value'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            WHEN nursingchartcelltypevlabel = 'Arterial Line MAP (mmHg)'
             AND nursingchartcelltypevalname = 'Value'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS ibp_mean
    FROM nursecharting
    -- speed up by only looking at a subset of charted data
    WHERE nursingchartcelltypecat IN (
        'Vital Signs',
        'Scores',
        'Other Vital Signs and Infusions'
    )
)
SELECT
    patientunitstayid,
    nursingchartoffset AS chartoffset,
    nursingchartentryoffset AS entryoffset,
    AVG(CASE WHEN heartrate >= 25 AND heartrate <= 225 THEN heartrate ELSE NULL END) AS heartrate,
    AVG(CASE WHEN respiratoryrate >= 0 AND respiratoryrate <= 60 THEN respiratoryrate ELSE NULL END) AS respiratoryrate,
    AVG(CASE WHEN o2saturation >= 0 AND o2saturation <= 100 THEN o2saturation ELSE NULL END) AS spo2,
    AVG(CASE WHEN nibp_systolic >= 25 AND nibp_systolic <= 250 THEN nibp_systolic ELSE NULL END) AS nibp_systolic,
    AVG(CASE WHEN nibp_diastolic >= 1 AND nibp_diastolic <= 200 THEN nibp_diastolic ELSE NULL END) AS nibp_diastolic,
    AVG(CASE WHEN nibp_mean >= 1 AND nibp_mean <= 250 THEN nibp_mean ELSE NULL END) AS nibp_mean,
    AVG(CASE WHEN temperature >= 25 AND temperature <= 46 THEN temperature ELSE NULL END) AS temperature,
    MAX(temperaturelocation) AS temperaturelocation,
    AVG(CASE WHEN ibp_systolic >= 1 AND ibp_systolic <= 300 THEN ibp_systolic ELSE NULL END) AS ibp_systolic,
    AVG(CASE WHEN ibp_diastolic >= 1 AND ibp_diastolic <= 200 THEN ibp_diastolic ELSE NULL END) AS ibp_diastolic,
    AVG(CASE WHEN ibp_mean >= 1 AND ibp_mean <= 250 THEN ibp_mean ELSE NULL END) AS ibp_mean
FROM nc
WHERE heartrate IS NOT NULL
   OR respiratoryrate IS NOT NULL
   OR o2saturation IS NOT NULL
   OR nibp_systolic IS NOT NULL
   OR nibp_diastolic IS NOT NULL
   OR nibp_mean IS NOT NULL
   OR temperature IS NOT NULL
   OR temperaturelocation IS NOT NULL
   OR ibp_systolic IS NOT NULL
   OR ibp_diastolic IS NOT NULL
   OR ibp_mean IS NOT NULL
GROUP BY patientunitstayid, nursingchartoffset, nursingchartentryoffset
ORDER BY patientunitstayid, nursingchartoffset, nursingchartentryoffset;
```

### 2. 提取 ICU 第一天的生命体征（宽表格式，基于 eicu-code）

```sql
-- 基于 eicu-code 官方实现，但简化为宽表格式（每个患者一行）
WITH nc AS (
    -- 同上，省略 CTE 定义
    SELECT
        patientunitstayid,
        nursingchartoffset,
        -- 提取各个生命体征（带异常值检查）
        CASE
            WHEN nursingchartcelltypevlabel = 'Heart Rate'
             AND nursingchartcelltypevalname = 'Heart Rate'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS heartrate,
        -- ... 省略其他指标的定义 ...
        CASE
            WHEN nursingchartcelltypevlabel = 'Temperature'
             AND nursingchartcelltypevalname = 'Temperature (C)'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS temperature
    FROM nursecharting
    WHERE nursingchartcelltypecat IN ('Vital Signs', 'Other Vital Signs and Infusions')
      AND nursingchartoffset >= 0 AND nursingchartoffset < 1440  -- 第一天
)
SELECT
    patientunitstayid,
    AVG(CASE WHEN heartrate >= 25 AND heartrate <= 225 THEN heartrate ELSE NULL END) AS heartrate_mean,
    MIN(CASE WHEN heartrate >= 25 AND heartrate <= 225 THEN heartrate ELSE NULL END) AS heartrate_min,
    MAX(CASE WHEN heartrate >= 25 AND heartrate <= 225 THEN heartrate ELSE NULL END) AS heartrate_max,
    AVG(CASE WHEN temperature >= 25 AND temperature <= 46 THEN temperature ELSE NULL END) AS temperature_mean,
    MIN(CASE WHEN temperature >= 25 AND temperature <= 46 THEN temperature ELSE NULL END) AS temperature_min,
    MAX(CASE WHEN temperature >= 25 AND temperature <= 46 THEN temperature ELSE NULL END) AS temperature_max
FROM nc
GROUP BY patientunitstayid
ORDER BY patientunitstayid;
```

### 3. 提取第一次生命体征记录

```sql
-- 提取每个患者的第一次生命体征记录（最小的 nursingchartoffset）
WITH first_vitals AS (
    SELECT
        nc.patientunitstayid,
        nc.nursingchartoffset,
        nc.heartrate,
        nc.temperature,
        nc.o2saturation,
        nc.nibp_systolic,
        nc.nibp_diastolic,
        nc.ibp_mean,
        ROW_NUMBER() OVER (
            PARTITION BY nc.patientunitstayid
            ORDER BY nc.nursingchartoffset
        ) AS rn
    FROM (
        -- 同上，省略 CTE 定义
        SELECT
            patientunitstayid,
            nursingchartoffset,
            CASE
                WHEN nursingchartcelltypevlabel = 'Heart Rate'
                 AND nursingchartcelltypevalname = 'Heart Rate'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
                ELSE NULL
            END AS heartrate,
            -- ... 省略其他指标 ...
        FROM nursecharting
        WHERE nursingchartcelltypecat IN ('Vital Signs', 'Other Vital Signs and Infusions')
          AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
    ) nc
)
SELECT *
FROM first_vitals
WHERE rn = 1
ORDER BY patientunitstayid;
```

---

## Python 模板

```python
import psycopg2
import pandas as pd
import numpy as np
import re

# 数据库连接
conn = psycopg2.connect(
    host="your_host",
    port=5432,
    dbname="eicu",
    user="your_user",
    password="your_password"
)

# nursingchartcelltypevlabel 和 nursingchartcelltypevalname 的准确值（基于 eicu-code）
VITAL_SIGNS_MAP = {
    'heartrate': {
        'label': 'Heart Rate',
        'valname': 'Heart Rate',
        'min_value': 25,
        'max_value': 225
    },
    'respiratoryrate': {
        'label': 'Respiratory Rate',
        'valname': 'Respiratory Rate',
        'min_value': 0,
        'max_value': 60
    },
    'o2saturation': {
        'label': 'O2 Saturation',
        'valname': 'O2 Saturation',
        'min_value': 0,
        'max_value': 100
    },
    'nibp_systolic': {
        'label': 'Non-Invasive BP',
        'valname': 'Non-Invasive BP Systolic',
        'min_value': 25,
        'max_value': 250
    },
    'nibp_diastolic': {
        'label': 'Non-Invasive BP',
        'valname': 'Non-Invasive BP Diastolic',
        'min_value': 1,
        'max_value': 200
    },
    'nibp_mean': {
        'label': 'Non-Invasive BP',
        'valname': 'Non-Invasive BP Mean',
        'min_value': 1,
        'max_value': 250
    },
    'temperature': {
        'label': 'Temperature',
        'valname': 'Temperature (C)',
        'min_value': 25,
        'max_value': 46
    },
    'ibp_systolic': {
        'label': 'Invasive BP',
        'valname': 'Invasive BP Systolic',
        'min_value': 1,
        'max_value': 300
    },
    'ibp_diastolic': {
        'label': 'Invasive BP',
        'valname': 'Invasive BP Diastolic',
        'min_value': 1,
        'max_value': 200
    },
    'ibp_mean': {
        'label': 'Invasive BP',
        'valname': 'Invasive BP Mean',
        'min_value': 1,
        'max_value': 250
    }
}

def is_numeric(value):
    """
    检查字符串是否为有效的数字
    
    Parameters:
    -----------
    value : str
        要检查的字符串
    
    Returns:
    --------
    bool : 是否为有效数字
    """
    if value is None or value in ('-', '.'):
        return False
    return bool(re.match(r'^[-]?[0-9]+[.]?[0-9]*$', value))

def extract_first_day_vitals_from_nursecharting(patient_ids=None, vitals_list=None):
    """
    从 nursecharting 表提取 ICU 第一天（0-1440 分钟）的生命体征
    （基于 eicu-code 官方实现）
    
    Parameters:
    -----------
    patient_ids : list or None
        患者 ID 列表，None 表示所有患者
    vitals_list : list or None
        要提取的生命体征列表，None 表示全部
    
    Returns:
    --------
    df : pandas DataFrame
        长格式生命体征数据
    """
    if vitals_list is None:
        vitals_list = list(VITAL_SIGNS_MAP.keys())
    
    # 构建 CTE（公用表表达式）
    case_clauses = []
    for vital_name in vitals_list:
        if vital_name not in VITAL_SIGNS_MAP:
            continue
        
        config = VITAL_SIGNS_MAP[vital_name]
        case_clause = f"""
        CASE
            WHEN nc.nursingchartcelltypevlabel = '{config['label']}'
             AND nc.nursingchartcelltypevalname = '{config['valname']}'
             AND nc.nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nc.nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nc.nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS {vital_name}
        """
        case_clauses.append(case_clause)
    
    if not case_clauses:
        raise ValueError("没有有效的生命体征配置")
    
    # 构建查询
    query = f"""
        WITH nc AS (
            SELECT
                nc.patientunitstayid,
                nc.nursingchartoffset,
                nc.nursingchartentryoffset,
                {','.join(case_clauses)}
            FROM nursecharting nc
            WHERE nc.nursingchartcelltypecat IN (
                'Vital Signs',
                'Scores',
                'Other Vital Signs and Infusions'
            )
              AND nc.nursingchartoffset >= 0 AND nc.nursingchartoffset < 1440
        )
        SELECT
            nc.patientunitstayid,
            nc.nursingchartoffset AS chartoffset,
            nc.nursingchartentryoffset AS entryoffset,
            {','.join([f"AVG(CASE WHEN {vital_name} >= {VITAL_SIGNS_MAP[vital_name]['min_value']} AND {vital_name} <= {VITAL_SIGNS_MAP[vital_name]['max_value']} THEN {vital_name} ELSE NULL END) AS {vital_name}" for vital_name in vitals_list if vital_name in VITAL_SIGNS_MAP])}
        FROM nc
        GROUP BY nc.patientunitstayid, nc.nursingchartoffset, nc.nursingchartentryoffset
        ORDER BY nc.patientunitstayid, nc.nursingchartoffset, nc.nursingchartentryoffset
    """
    
    # 添加患者 ID 过滤
    if patient_ids is not None:
        if len(patient_ids) == 1:
            query = query.replace(
                "WHERE nc.nursingchartcelltypecat",
                f"WHERE nc.patientunitstayid = {patient_ids[0]} AND nc.nursingchartcelltypecat"
            )
        else:
            # 使用临时表
            ids_str = ",".join(map(str, patient_ids))
            query = f"""
                WITH target_patients AS (
                    SELECT unnest(ARRAY[{ids_str}]) AS pid
                )
                {query}
            """
            query = query.replace(
                "FROM nc",
                "FROM nc, target_patients tp WHERE nc.patientunitstayid = tp.pid"
            )
    
    # 执行查询
    df = pd.read_sql_query(query, conn)
    
    return df

def aggregate_vitals_to_wide(df):
    """
    将长格式生命体征数据转换为宽表格式
    
    Parameters:
    -----------
    df : pandas DataFrame
        长格式数据（来自 extract_first_day_vitals_from_nursecharting）
    
    Returns:
    --------
    wide_df : pandas DataFrame
        宽表格式，每个患者一行
    """
    # 计算统计量
    agg_df = df.groupby('patientunitstayid').agg(
        {col: ['mean', 'min', 'max', 'count'] for col in df.columns if col not in ['patientunitstayid', 'chartoffset', 'entryoffset']}
    ).reset_index()
    
    # 展平列名
    agg_df.columns = ['_'.join(col).strip() if col[1] else col[0] for col in agg_df.columns.values]
    
    return agg_df

# 使用示例
if __name__ == "__main__":
    # 示例 1：提取所有患者的心率和体温
    vitals = extract_first_day_vitals_from_nursecharting(
        vitals_list=['heartrate', 'temperature']
    )
    print(f"提取了 {len(vitals)} 条记录")
    print(vitals.head(10))
    
    # 示例 2：提取特定患者的所有生命体征
    vitals = extract_first_day_vitals_from_nursecharting(
        patient_ids=[154347, 112001, 189456]
    )
    
    # 示例 3：转换为宽表
    wide_df = aggregate_vitals_to_wide(vitals)
    print("\n宽表格式:")
    print(wide_df.head(5))
    
    conn.close()
```

---

## 注意事项

### 1. 使用精确匹配（重要！）

✅ **正确** (精确匹配):
```sql
WHERE nc.nursingchartcelltypevlabel = 'Heart Rate'
  AND nc.nursingchartcelltypevalname = 'Heart Rate'
```

❌ **错误** (模糊匹配):
```sql
WHERE nc.nursingchartcelltypevlabel ILIKE '%Heart%'
```

### 2. 异常值检查（重要！）

基于 eicu-code 官方实现，查询时务必添加异常值检查：

```sql
AVG(CASE WHEN heartrate >= 25 AND heartrate <= 225 THEN heartrate ELSE NULL END) AS heartrate
```

### 3. 数据类型转换

- `nursingchartvalue` 字段类型为 `VARCHAR`
- 计算时需转换：`CAST(nc.nursingchartvalue AS NUMERIC)` 或 `nc.nursingchartvalue::NUMERIC`
- **务必先检查是否为有效数字**：`nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'`

### 4. 性能优化

- `nursecharting` 表有 **1.51 亿条记录**
- 查询时务必:
  - 使用 `patientunitstayid` 过滤
  - 限定 `nursingchartoffset` 范围
  - 使用 `nursingchartcelltypecat` 过滤（如 `'Vital Signs'`）
  - 添加 `LIMIT` 子句（测试时）
  - 使用 `EXPLAIN` 分析查询计划

### 5. 时间偏移量

- `nursingchartoffset`: 记录时间（相对于 ICU 入住的分钟数）
- `nursingchartentryoffset`: 录入时间（相对于 ICU 入住的分钟数）
- 第一天 = `nursingchartoffset >= 0 AND nursingchartoffset < 1440`

---

## 参考链接

- **nursecharting 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/nursecharting.html
- **eicu-code 官方实现**: `D:\Project\eicu-code\concepts\pivoted\pivoted-vital.sql`
- **eICU 数据介绍**: https://eicu-crd.mit.edu/

---

**最后更新**: 2026-05-22  
**更新人**: 悟空（基于 eicu-code 官方实现）
