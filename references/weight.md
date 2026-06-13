# eICU 2.0 体重（Weight）查询

**参考**: 
- eicu-code 官方实现: `D:\Project\eicu-code\concepts\pivoted\pivoted-weight.sql`
- eICU 2.0 SchemaSpy 文档: https://lcp.mit.edu/eicu-schema-spy/

---

## 目录

1. [为什么体重数据复杂？](#为什么体重数据复杂)
2. [数据来源](#数据来源)
3. [SQL 模板](#sql-模板)
4. [Python 模板](#python-模板)
5. [注意事项](#注意事项)

---

## 为什么体重数据复杂？

体重数据在 eICU 中**分散在多个表**中：

1. **`patient` 表**: 入院体重（`admissionweight`）、入院身高（`admissionheight`）
2. **`nursecharting` 表**: 护理记录中的体重（如 "Admission Weight"、"WEIGHT in Kg"）
3. **`intakeoutput` 表**: 入出量记录中的体重（如 "Bodyweight (kg)"、"Bodyweight (lb)"）
4. **`infusiondrug` 表**: 输液记录中的患者体重（`patientweight`）

eicu-code **合并所有来源**，并进行了数据清洗（如身高体重互换检查、单位转换等）。

---

## 数据来源

### 1. `patient` 表

| 字段 | 说明 | 单位 |
|------|------|------|
| `admissionweight` | 入院体重 | kg |
| `admissionheight` | 入院身高 | cm |
| `dischargeweight` | 出院体重 | kg |

**数据清洗**（来自 eicu-code）:
- 检查身高体重是否互换（`admissionweight >= 100 AND admissionheight <= 100 AND abs(admissionheight-admissionweight) >= 20` → 可能是互换）
- 身高转换：
  - 如果 `height <= 0.30` → NULL（无效）
  - 如果 `height <= 2.5` → `height*100`（可能是米，转换为厘米）
  - 如果 `height <= 10` → NULL（可能是分米，但 eICU 中不太可能）
  - 如果 `height <= 25` → `height*10`（可能是分米，转换为厘米）
  - 如果 `height > 250` → NULL（无效）
  - 否则 → `height`（已经是厘米）
- 体重检查：
  - 如果 `weight <= 20` → NULL（无效）
  - 如果 `weight > 300` → NULL（无效）
  - 否则 → `weight`（已经是 kg）

### 2. `nursecharting` 表

| nursingchartcelltypevlabel | 说明 |
|----------------------------|------|
| `'Admission Weight'` | 入院体重 |
| `'Admit weight'` | 入院体重 |
| `'WEIGHT in Kg'` | 体重（kg） |

**提取逻辑**:
```sql
SELECT 
    patientunitstayid,
    nursingchartoffset AS chartoffset,
    CASE 
        WHEN nursingchartcelltypevlabel IN ('Admission Weight', 'Admit weight') 
        THEN 'admit'
        ELSE 'daily' 
    END AS weight_type,
    CAST(nursingchartvalue AS NUMERIC) AS weight
FROM nursecharting
WHERE nursingchartcelltypecat = 'Other Vital Signs and Infusions'
  AND nursingchartcelltypevlabel IN ('Admission Weight', 'Admit weight', 'WEIGHT in Kg')
  -- 确保 nursingchartvalue 是数字
  AND REGEXP_CONTAINS(nursingchartvalue, r'^([0-9]+\.?[0-9]*|\.[0-9]+)$')
  -- 只取 ICU 第一天
  AND nursingchartoffset < 60*24
```

### 3. `intakeoutput` 表

| cellpath | 说明 | 单位 |
|----------|------|------|
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Weight\|Bodyweight (kg)'` | 体重 | kg |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Weight\|Bodyweight (lb)'` | 体重 | lb（需转换为 kg） |

**提取逻辑**:
```sql
SELECT 
    patientunitstayid,
    intakeoutputoffset AS chartoffset,
    'daily' AS weight_type,
    MAX(CASE 
            WHEN cellpath = 'flowsheet|Flowsheet Cell Labels|I&O|Weight|Bodyweight (kg)' 
            THEN cellvaluenumeric 
        END) AS weight_kg,
    -- 约 300 条记录使用 lb，需转换
    MAX(CASE 
            WHEN cellpath = 'flowsheet|Flowsheet Cell Labels|I&O|Weight|Bodyweight (lb)' 
            THEN cellvaluenumeric * 0.453592  -- lb 转 kg
        END) AS weight_kg2
FROM intakeoutput
WHERE cellpath IN (
    'flowsheet|Flowsheet Cell Labels|I&O|Weight|Bodyweight (kg)',
    'flowsheet|Flowsheet Cell Labels|I&O|Weight|Bodyweight (lb)'
)
  -- 只取 ICU 第一天
  AND intakeoutputoffset < 60*24
GROUP BY patientunitstayid, intakeoutputoffset
```

### 4. `infusiondrug` 表

| 字段 | 说明 | 单位 |
|------|------|------|
| `patientweight` | 患者体重 | kg |

**提取逻辑**:
```sql
SELECT 
    patientunitstayid,
    infusionoffset AS chartoffset,
    'daily' AS weight_type,
    CAST(patientweight AS NUMERIC) AS weight
FROM infusiondrug
WHERE patientweight IS NOT NULL
  -- 只取 ICU 第一天
  AND infusionoffset < 60*24
```

---

## SQL 模板

### 1. 提取 ICU 第一天的体重（长格式，基于 eicu-code）

```sql
-- 基于 eicu-code 官方实现：D:\Project\eicu-code\concepts\pivoted\pivoted-weight.sql
-- 提取 ICU 第一天（0-1440 分钟）的体重（长格式，合并所有来源）

-- 来源 1: patient 表（入院体重）
WITH htwt AS (
    SELECT
        patientunitstayid,
        hospitaladmitoffset AS chartoffset,
        admissionheight AS height,
        admissionweight AS weight,
        CASE
            -- 检查身高体重是否互换
            WHEN admissionweight >= 100
             AND admissionheight > 25 AND admissionheight <= 100
             AND ABS(admissionheight - admissionweight) >= 20
            THEN 'swap'
        END AS method
    FROM patient
),
htwt_fixed AS (
    SELECT
        patientunitstayid,
        chartoffset,
        'admit' AS weight_type,
        CASE
            WHEN method = 'swap' THEN weight
            WHEN height <= 0.30 THEN NULL
            WHEN height <= 2.5 THEN height * 100  -- 可能是米，转换为厘米
            WHEN height <= 10 THEN NULL         -- 可能是分米，但 eICU 中不太可能
            WHEN height <= 25 THEN height * 10  -- 可能是分米，转换为厘米
            WHEN height > 250 THEN NULL        -- 无效
            ELSE height
        END AS height_fixed,
        CASE
            WHEN method = 'swap' THEN height
            WHEN weight <= 20 THEN NULL        -- 无效
            WHEN weight > 300 THEN NULL       -- 无效
            ELSE weight
        END AS weight_fixed
    FROM htwt
),
wt1 AS (
    -- 来源 2: nursecharting 表
    SELECT
        patientunitstayid,
        nursingchartoffset AS chartoffset,
        CASE 
            WHEN nursingchartcelltypevlabel IN ('Admission Weight', 'Admit weight') 
            THEN 'admit'
            ELSE 'daily' 
        END AS weight_type,
        CAST(nursingchartvalue AS NUMERIC) AS weight
    FROM nursecharting
    WHERE nursingchartcelltypecat = 'Other Vital Signs and Infusions'
      AND nursingchartcelltypevlabel IN ('Admission Weight', 'Admit weight', 'WEIGHT in Kg')
      -- 确保 nursingchartvalue 是数字
      AND REGEXP_CONTAINS(nursingchartvalue, r'^([0-9]+\.?[0-9]*|\.[0-9]+)$')
      -- 只取 ICU 第一天
      AND nursingchartoffset < 60*24
),
wt2 AS (
    -- 来源 3: intakeoutput 表
    SELECT
        patientunitstayid,
        intakeoutputoffset AS chartoffset,
        'daily' AS weight_type,
        MAX(CASE 
                WHEN cellpath = 'flowsheet|Flowsheet Cell Labels|I&O|Weight|Bodyweight (kg)' 
                THEN cellvaluenumeric 
            END) AS weight_kg,
        -- 约 300 条记录使用 lb，需转换
        MAX(CASE 
                WHEN cellpath = 'flowsheet|Flowsheet Cell Labels|I&O|Weight|Bodyweight (lb)' 
                THEN cellvaluenumeric * 0.453592  -- lb 转 kg
            END) AS weight_kg2
    FROM intakeoutput
    WHERE cellpath IN (
        'flowsheet|Flowsheet Cell Labels|I&O|Weight|Bodyweight (kg)',
        'flowsheet|Flowsheet Cell Labels|I&O|Weight|Bodyweight (lb)'
    )
      -- 只取 ICU 第一天
      AND intakeoutputoffset < 60*24
    GROUP BY patientunitstayid, intakeoutputoffset
),
wt3 AS (
    -- 来源 4: infusiondrug 表
    SELECT
        patientunitstayid,
        infusionoffset AS chartoffset,
        'daily' AS weight_type,
        CAST(patientweight AS NUMERIC) AS weight
    FROM infusiondrug
    WHERE patientweight IS NOT NULL
      -- 只取 ICU 第一天
      AND infusionoffset < 60*24
)
-- 合并所有来源
SELECT 
    patientunitstayid,
    chartoffset,
    'patient' AS source_table,
    weight_type,
    weight_fixed AS weight
FROM htwt_fixed
WHERE weight_fixed IS NOT NULL

UNION ALL

SELECT 
    patientunitstayid,
    chartoffset,
    'nursecharting' AS source_table,
    weight_type,
    weight
FROM wt1
WHERE weight IS NOT NULL

UNION ALL

SELECT 
    patientunitstayid,
    chartoffset,
    'intakeoutput' AS source_table,
    weight_type,
    COALESCE(weight_kg, weight_kg2) AS weight
FROM wt2
WHERE weight_kg IS NOT NULL OR weight_kg2 IS NOT NULL

UNION ALL

SELECT 
    patientunitstayid,
    chartoffset,
    'infusiondrug' AS source_table,
    weight_type,
    weight
FROM wt3
WHERE weight IS NOT NULL

ORDER BY patientunitstayid, chartoffset, source_table;
```

### 2. 提取 ICU 第一天的体重（宽表格式）

```sql
-- 提取 ICU 第一天（0-1440 分钟）的体重（宽表格式，每个患者一行）
WITH all_weights AS (
    -- 同上，省略 CTE 定义
    -- ... (同上) ...
)
SELECT
    patientunitstayid,
    AVG(weight) AS weight_mean,        -- 平均体重
    MIN(weight) AS weight_min,         -- 最小体重
    MAX(weight) AS weight_max,         -- 最大体重
    COUNT(*) AS num_records,           -- 记录数
    MAX(CASE WHEN weight_type = 'admit' THEN weight ELSE NULL END) AS admission_weight,  -- 入院体重
    MAX(height_fixed) AS admission_height  -- 入院身高（来自 patient 表）
FROM all_weights
WHERE weight IS NOT NULL
GROUP BY patientunitstayid
ORDER BY patientunitstayid;
```

---

## Python 模板

```python
import psycopg2
import pandas as pd
import numpy as np

# 数据库连接
conn = psycopg2.connect(
    host="your_host",
    port=5432,
    dbname="eicu",
    user="your_user",
    password="your_password"
)

def extract_first_day_weight(patient_ids=None):
    """
    从多个表提取 ICU 第一天（0-1440 分钟）的体重
    （基于 eicu-code 官方实现）
    
    Parameters:
    -----------
    patient_ids : list or None
        患者 ID 列表，None 表示所有患者
    
    Returns:
    --------
    df : pandas DataFrame
        长格式体重数据（包含所有来源）
    """
    # 构建查询（省略完整的 CTE 定义，实际使用时应包含）
    query = """
        -- 来源 1: patient 表（入院体重）
        WITH htwt AS (
            SELECT
                patientunitstayid,
                hospitaladmitoffset AS chartoffset,
                admissionheight AS height,
                admissionweight AS weight,
                CASE
                    WHEN admissionweight >= 100
                     AND admissionheight > 25 AND admissionheight <= 100
                     AND ABS(admissionheight - admissionweight) >= 20
                    THEN 'swap'
                END AS method
            FROM patient
        ),
        htwt_fixed AS (
            SELECT
                patientunitstayid,
                chartoffset,
                'admit' AS weight_type,
                CASE
                    WHEN method = 'swap' THEN weight
                    WHEN height <= 0.30 THEN NULL
                    WHEN height <= 2.5 THEN height * 100
                    WHEN height <= 10 THEN NULL
                    WHEN height <= 25 THEN height * 10
                    WHEN height > 250 THEN NULL
                    ELSE height
                END AS height_fixed,
                CASE
                    WHEN method = 'swap' THEN height
                    WHEN weight <= 20 THEN NULL
                    WHEN weight > 300 THEN NULL
                    ELSE weight
                END AS weight_fixed
            FROM htwt
        ),
        -- ... 省略其他 CTE 定义 ...
        all_weights AS (
            -- 合并所有来源
            SELECT 
                patientunitstayid,
                chartoffset,
                'patient' AS source_table,
                weight_type,
                weight_fixed AS weight
            FROM htwt_fixed
            WHERE weight_fixed IS NOT NULL
            
            UNION ALL
            
            -- ... 省略其他 UNION ALL ...
        )
        SELECT *
        FROM all_weights
        ORDER BY patientunitstayid, chartoffset, source_table
    """
    
    # 添加患者 ID 过滤
    if patient_ids is not None:
        if len(patient_ids) == 1:
            query = query.replace(
                "WHERE weight_fixed",
                f"WHERE patientunitstayid = {patient_ids[0]} AND weight_fixed"
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
                "FROM patient",
                "FROM patient, target_patients tp WHERE patientunitstayid = tp.pid"
            )
    
    # 执行查询
    df = pd.read_sql_query(query, conn)
    
    return df

def aggregate_weight_to_wide(df):
    """
    将长格式体重数据转换为宽表格式
    
    Parameters:
    -----------
    df : pandas DataFrame
        长格式数据（来自 extract_first_day_weight）
    
    Returns:
    --------
    wide_df : pandas DataFrame
        宽表格式，每个患者一行
    """
    # 计算统计量
    agg_df = df.groupby('patientunitstayid').agg(
        weight_mean=('weight', 'mean'),
        weight_min=('weight', 'min'),
        weight_max=('weight', 'max'),
        num_records=('weight', 'count'),
        admission_weight=('weight', lambda x: x[df.loc[x.index, 'weight_type'] == 'admit'].iloc[0] if any(df.loc[x.index, 'weight_type'] == 'admit') else np.nan)
    ).reset_index()
    
    # 添加入院身高（来自 patient 表）
    height_df = df[df['source_table'] == 'patient'][['patientunitstayid', 'height_fixed']].drop_duplicates()
    height_df = height_df.rename(columns={'height_fixed': 'admission_height'})
    agg_df = agg_df.merge(height_df, on='patientunitstayid', how='left')
    
    return agg_df

# 使用示例
if __name__ == "__main__":
    # 示例 1：提取所有患者的体重
    weight = extract_first_day_weight()
    print(f"提取了 {len(weight)} 条记录")
    print(weight.head(10))
    
    # 示例 2：提取特定患者的体重
    weight = extract_first_day_weight(
        patient_ids=[154347, 112001, 189456]
    )
    
    # 示例 3：转换为宽表
    wide_df = aggregate_weight_to_wide(weight)
    print("\n宽表格式:")
    print(wide_df.head(5))
    
    conn.close()
```

---

## 注意事项

### 1. 数据来源多样（重要！）

✅ **正确**: 从**多个表**提取体重，并合并结果。

❌ **错误**: 只从 `patient` 表提取 `admissionweight`，会丢失大量数据。

### 2. 数据清洗（重要！）

eicu-code 进行了以下数据清洗：

1. **身高体重互换检查**: 
   ```sql
   WHEN admissionweight >= 100
    AND admissionheight > 25 AND admissionheight <= 100
    AND ABS(admissionheight - admissionweight) >= 20
   THEN 'swap'
   ```

2. **身高单位转换**:
   - 如果 `height <= 2.5` → 可能是米，转换为厘米（`height*100`）
   - 如果 `height <= 25` → 可能是分米，转换为厘米（`height*10`）

3. **体重单位转换**:
   - 如果 `cellpath` 包含 `'Bodyweight (lb)'` → 需转换（`cellvaluenumeric * 0.453592`）

4. **异常值检查**:
   - 身高：`height <= 0.30` 或 `height > 250` → NULL
   - 体重：`weight <= 20` 或 `weight > 300` → NULL

### 3. 性能优化

- 查询涉及 **4 个大表**（`patient`、`nursecharting`、`intakeoutput`、`infusiondrug`）
- 查询时务必:
  - 使用 `patientunitstayid` 过滤
  - 限定时间范围（如 `nursingchartoffset < 60*24`）
  - 添加 `LIMIT` 子句（测试时）
  - 使用 `EXPLAIN` 分析查询计划

### 4. 时间偏移量

- 不同表使用不同的时间字段：
  - `patient` 表：`hospitaladmitoffset`
  - `nursecharting` 表：`nursingchartoffset`
  - `intakeoutput` 表：`intakeoutputoffset`
  - `infusiondrug` 表：`infusionoffset`
- 第一天 = `offset >= 0 AND offset < 1440`

---

## 常用查询场景

### 1. 计算 AKI 的尿量阈值（需要体重）

```sql
-- 计算 AKI 的尿量阈值（0.5 ml/kg/h）
WITH weight_data AS (
    -- 省略：提取患者体重（使用 admission_weight 或平均体重）
    SELECT
        patientunitstayid,
        AVG(weight) AS weight_mean
    FROM (
        -- 省略：合并所有体重来源
        -- ... (同上) ...
    ) all_weights
    WHERE weight IS NOT NULL
    GROUP BY patientunitstayid
),
uo_data AS (
    -- 省略：提取患者尿量
    -- ... (来自 urion_output.md) ...
)
SELECT
    wd.patientunitstayid,
    wd.weight_mean,
    ud.urine_output_total,
    ud.urine_output_total / (wd.weight_mean * 24) AS urine_output_per_kg_per_hour
FROM weight_data wd
LEFT JOIN uo_data ud ON wd.patientunitstayid = ud.patientunitstayid
WHERE ud.urine_output_total / (wd.weight_mean * 24) < 0.5  -- AKI 少尿阈值
ORDER BY wd.patientunitstayid;
```

### 2. 计算 BMI

```sql
-- 计算 BMI（需要身高和体重）
WITH weight_data AS (
    -- 省略：提取患者体重
    -- ... (同上) ...
),
height_data AS (
    -- 省略：提取患者身高（来自 patient 表）
    SELECT
        patientunitstayid,
        MAX(height_fixed) AS height_cm
    FROM (
        -- 省略：patient 表的身高数据
        -- ... (同上) ...
    ) all_heights
    WHERE height_fixed IS NOT NULL
    GROUP BY patientunitstayid
)
SELECT
    wd.patientunitstayid,
    wd.weight_mean,
    hd.height_cm,
    wd.weight_mean / POWER(hd.height_cm/100, 2) AS bmi
FROM weight_data wd
LEFT JOIN height_data hd ON wd.patientunitstayid = hd.patientunitstayid
WHERE hd.height_cm IS NOT NULL
ORDER BY wd.patientunitstayid;
```

---

## 参考链接

- **patient 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/patient.html
- **nursecharting 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/nursecharting.html
- **intakeoutput 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/intakeoutput.html
- **infusiondrug 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/infusiondrug.html
- **eicu-code 官方实现**: `D:\Project\eicu-code\concepts\pivoted\pivoted-weight.sql`
- **eICU 数据介绍**: https://eicu-crd.mit.edu/

---

**最后更新**: 2026-05-22  
**更新人**: 悟空（基于 eicu-code 官方实现）
