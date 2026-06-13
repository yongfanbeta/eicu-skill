# eICU 2.0 血气分析查询

**参考**: 
- eICU 2.0 SchemaSpy 文档: https://lcp.mit.edu/eicu-schema-spy/
- `lab` 表（血气指标通常在此表中）

---

## 目录

1. [常用血气指标映射](#常用血气指标映射)
2. [SQL 模板](#sql-模板)
3. [Python 模板](#python-模板)
4. [注意事项](#注意事项)

---

## 常用血气指标映射

> **重要**: 血气指标通常存储在 `lab` 表中，使用**精确匹配**（`labname = 'xxx'`）。

| 指标 | 准确 labname | 单位 | 正常范围 | 说明 |
|------|-------------|------|----------|------|
| pH | `'pH'` | 无 | 7.35-7.45 | 酸碱度 |
| 动脉血 pH | `'pH (ART)'` | 无 | 7.35-7.45 | 动脉血 |
| 氧分压 | `'pO2'` | mmHg | 80-100 | 动脉血氧分压 |
| 动脉血氧分压 | `'pO2 (ART)'` | mmHg | 80-100 | 动脉血 |
| 二氧化碳分压 | `'pCO2'` | mmHg | 35-45 | 动脉血二氧化碳分压 |
| 动脉血二氧化碳分压 | `'pCO2 (ART)'` | mmHg | 35-45 | 动脉血 |
| 氧饱和度 | `'O2 saturation'` | % | 95-100 | 血氧饱和度 |
| 动脉血氧饱和度 | `'O2 sat (ART)'` | % | 95-100 | 动脉血 |
| 碳酸氢根 | `'bicarbonate'` 或 `'HCO3'` | mEq/L | 22-26 |  |
| 乳酸 | `'lactate'` | mmol/L | 0.5-1.0 | 组织氧合指标 |
| 碱剩余 | `'base excess'` | mEq/L | -2 to +2 |  |
| 阴离子间隙 | `'anion gap'` | mEq/L | 8-12 |  |

⚠️ **注意**: 不同 ICU 站点可能使用不同的 `labname`（如 `'pH'` vs `'pH (ART)'`）。建议先查询确认：

```sql
-- 查看所有血气相关的 labname
SELECT DISTINCT labname, COUNT(*) AS cnt
FROM lab
WHERE labname IN (
  'pH',
  'pH (ART)',
  'pO2',
  'pO2 (ART)',
  'pCO2',
  'pCO2 (ART)',
  'O2 saturation',
  'O2 sat (ART)',
  'bicarbonate',
  'HCO3',
  'lactate',
  'base excess',
  'anion gap'
)
GROUP BY labname
ORDER BY cnt DESC;
```

---

## SQL 模板

### 1. 提取第一次血气分析（基于 eicu-code 逻辑）

```sql
-- 提取每个患者的第一次血气分析（最小的 offset）
SELECT 
    l.patientunitstayid,
    l.labname,
    l.labresultoffset,
    l.labresult,
    l.labmeasurenamesystem,
    l.labmeasurenameinterface
FROM lab l
WHERE l.labresultoffset >= 0 AND l.labresultoffset < 1440  -- 第一天
  AND l.labresult IS NOT NULL
  AND l.labname IN (
    'pH',
    'pH (ART)',
    'pO2',
    'pO2 (ART)',
    'pCO2',
    'pCO2 (ART)',
    'O2 saturation',
    'O2 sat (ART)',
    'bicarbonate',
    'HCO3',
    'lactate',
    'base excess',
    'anion gap'
  )
  AND l.labresultoffset = (
    SELECT MIN(l2.labresultoffset)
    FROM lab l2
    WHERE l2.patientunitstayid = l.patientunitstayid
      AND l2.labname = l.labname
      AND l2.labresultoffset >= 0 AND l2.labresultoffset < 1440
  )
ORDER BY l.patientunitstayid, l.labname;
```

### 2. 提取第一天所有血气指标（宽表格式）

```sql
-- 基于 eicu-code 逻辑：使用精确匹配 + 异常值检查
SELECT
  pvt.patientunitstayid,
  min(CASE WHEN labname = 'pH' THEN labresult ELSE null END) as ph_min,
  max(CASE WHEN labname = 'pH' THEN labresult ELSE null END) as ph_max,
  min(CASE WHEN labname = 'pO2' THEN labresult ELSE null END) as po2_min,
  max(CASE WHEN labname = 'pO2' THEN labresult ELSE null END) as po2_max,
  min(CASE WHEN labname = 'pCO2' THEN labresult ELSE null END) as pco2_min,
  max(CASE WHEN labname = 'pCO2' THEN labresult ELSE null END) as pco2_max,
  min(CASE WHEN labname = 'O2 saturation' THEN labresult ELSE null END) as o2_sat_min,
  max(CASE WHEN labname = 'O2 saturation' THEN labresult ELSE null END) as o2_sat_max,
  min(CASE WHEN labname = 'bicarbonate' THEN labresult ELSE null END) as bicarbonate_min,
  max(CASE WHEN labname = 'bicarbonate' THEN labresult ELSE null END) as bicarbonate_max,
  min(CASE WHEN labname = 'lactate' THEN labresult ELSE null END) as lactate_min,
  max(CASE WHEN labname = 'lactate' THEN labresult ELSE null END) as lactate_max
FROM
( 
  SELECT l.patientunitstayid, l.labname
  , CASE
      WHEN l.labname = 'pH' AND (l.labresult < 6.8 OR l.labresult > 7.8) THEN null  -- 异常 pH 值
      WHEN l.labname = 'pO2' AND (l.labresult < 0 OR l.labresult > 700) THEN null  -- 异常 pO2 值
      WHEN l.labname = 'pCO2' AND (l.labresult < 0 OR l.labresult > 150) THEN null  -- 异常 pCO2 值
      WHEN l.labname = 'O2 saturation' AND (l.labresult < 0 OR l.labresult > 100) THEN null  -- 异常 O2 sat 值
      WHEN l.labname = 'bicarbonate' AND l.labresult > 10000 THEN null  -- 同 labs.md 中的异常值检查
      WHEN l.labname = 'lactate' AND l.labresult > 50 THEN null  -- 同 labs.md 中的异常值检查
      ELSE l.labresult
    END AS labresult
  FROM lab l
  WHERE l.labresultoffset >= 0 AND l.labresultoffset < 1440  -- 第一天
    AND l.labresult IS NOT NULL
    AND l.labname IN (
      'pH',
      'pO2',
      'pCO2',
      'O2 saturation',
      'bicarbonate',
      'lactate'
    )
) pvt
GROUP BY pvt.patientunitstayid
ORDER BY pvt.patientunitstayid;
```

### 3. 筛选异常血气值（如 pH < 7.35 或乳酸 > 2.0）

```sql
-- 筛选酸中毒患者（pH < 7.35 或乳酸 > 2.0）
SELECT DISTINCT l.patientunitstayid
FROM lab l
WHERE l.labresultoffset >= 0 AND l.labresultoffset < 1440  -- 第一天
  AND (
    (l.labname = 'pH' AND l.labresult::numeric < 7.35)
    OR
    (l.labname = 'lactate' AND l.labresult::numeric > 2.0)
  )
ORDER BY l.patientunitstayid;
```

### 4. 计算氧合指数（OI = pO2 / FiO2）

⚠️ **注意**: 计算 OI 需要同时获取 `pO2`（来自 `lab` 表）和 `FiO2`（通常来自 `respiratory` 表或 `nursecharting` 表）。

```sql
-- 简化版本：仅使用 lab 表中的 pO2（假设 FiO2 = 1.0 或已知）
SELECT 
    l.patientunitstayid,
    l.labresultoffset,
    l.labresult AS po2,
    -- 如果知道 FiO2，可以计算 OI = pO2 / FiO2
    -- 例如：l.labresult::numeric / 0.5 AS oi  -- 假设 FiO2 = 50%
    l.labresult::numeric AS oi_assuming_fio2_1  -- 假设 FiO2 = 1.0
FROM lab l
WHERE l.labresultoffset >= 0 AND l.labresultoffset < 1440
  AND l.labname = 'pO2'
  AND l.labresult IS NOT NULL
ORDER BY l.patientunitstayid, l.labresultoffset;
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

# 常用血气指标
BLOOD_GAS_LABNAMES = [
    'pH',
    'pH (ART)',
    'pO2',
    'pO2 (ART)',
    'pCO2',
    'pCO2 (ART)',
    'O2 saturation',
    'O2 sat (ART)',
    'bicarbonate',
    'HCO3',
    'lactate',
    'base excess',
    'anion gap'
]

# 异常值检查阈值
BLOOD_GAS_SANITY = {
    'pH': lambda x: 6.8 <= x <= 7.8,
    'pO2': lambda x: 0 <= x <= 700,  # mmHg
    'pCO2': lambda x: 0 <= x <= 150,  # mmHg
    'O2 saturation': lambda x: 0 <= x <= 100,  # %
    'bicarbonate': lambda x: 0 <= x <= 10000,  # mEq/L
    'lactate': lambda x: 0 <= x <= 50  # mmol/L
}

def extract_first_day_blood_gas(patient_ids=None, lab_names=None):
    """
    提取 ICU 第一天（0-1440 分钟）的血气分析指标
    
    Parameters:
    -----------
    patient_ids : list or None
        患者 ID 列表，None 表示所有患者
    lab_names : list or None
        要提取的血气指标，None 表示全部（使用 BLOOD_GAS_LABNAMES）
    
    Returns:
    --------
    df : pandas DataFrame
        长格式血气数据
    """
    if lab_names is None:
        lab_names = BLOOD_GAS_LABNAMES
    
    # 构建 labname 过滤条件（使用精确匹配）
    lab_names_str = "','".join(lab_names)
    
    # 构建查询
    query = f"""
        SELECT 
            l.patientunitstayid,
            l.labname,
            l.labresultoffset,
            l.labresult,
            CASE
        """
    
    # 添加异常值检查
    case_clauses = []
    for lab_name, check_func in BLOOD_GAS_SANITY.items():
        if lab_name in lab_names:
            if lab_name == 'pH':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND (l.labresult < 6.8 OR l.labresult > 7.8) THEN NULL")
            elif lab_name == 'pO2':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND (l.labresult < 0 OR l.labresult > 700) THEN NULL")
            elif lab_name == 'pCO2':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND (l.labresult < 0 OR l.labresult > 150) THEN NULL")
            elif lab_name == 'O2 saturation':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND (l.labresult < 0 OR l.labresult > 100) THEN NULL")
            elif lab_name == 'bicarbonate':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 10000 THEN NULL")
            elif lab_name == 'lactate':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 50 THEN NULL")
    
    if case_clauses:
        query += "\n".join(case_clauses) + "\n      ELSE l.labresult\n    END AS labresult_checked,\n"
    else:
        query += "      l.labresult AS labresult_checked,\n"
    
    query += f"""
            l.labmeasurenamesystem,
            l.labmeasurenameinterface
        FROM lab l
        WHERE l.labresultoffset >= 0 AND l.labresultoffset < 1440
          AND l.labresult IS NOT NULL
          AND l.labname IN ('{lab_names_str}')
          AND l.labresult > 0  -- lab values cannot be 0 and cannot be negative
        ORDER BY l.patientunitstayid, l.labname, l.labresultoffset
    """
    
    # 添加患者 ID 过滤
    if patient_ids is not None:
        if len(patient_ids) == 1:
            query = query.replace(
                "WHERE l.labresultoffset",
                f"WHERE l.patientunitstayid = {patient_ids[0]} AND l.labresultoffset"
            )
        else:
            # 使用临时表
            ids_str = ",".join(map(str, patient_ids))
            query = f"""
                WITH target_patients AS (
                    SELECT unnest(ARRAY[{ids_str}]) AS pid
                )
                SELECT 
                    l.patientunitstayid,
                    l.labname,
                    l.labresultoffset,
                    l.labresult,
                    CASE
            """
            # 重新添加异常值检查
            if case_clauses:
                query += "\n".join(case_clauses) + "\n      ELSE l.labresult\n    END AS labresult_checked,\n"
            else:
                query += "      l.labresult AS labresult_checked,\n"
            
            query += f"""
                        l.labmeasurenamesystem,
                        l.labmeasurenameinterface
                FROM lab l, target_patients tp
                WHERE l.patientunitstayid = tp.pid
                  AND l.labresultoffset >= 0 AND l.labresultoffset < 1440
                  AND l.labresult IS NOT NULL
                  AND l.labname IN ('{lab_names_str}')
                  AND l.labresult > 0
                ORDER BY l.patientunitstayid, l.labname, l.labresultoffset
            """
    
    # 执行查询
    df = pd.read_sql_query(query, conn)
    
    # 转换 labresult 为数值（如果可能）
    df['labresult_numeric'] = pd.to_numeric(df['labresult_checked'], errors='coerce')
    
    return df

def pivot_blood_gas_to_wide(df):
    """
    将长格式血气数据转换为宽表格式
    
    Parameters:
    -----------
    df : pandas DataFrame
        长格式数据（来自 extract_first_day_blood_gas）
    
    Returns:
    --------
    wide_df : pandas DataFrame
        宽表格式，每个患者一行，包含各指标的 min 和 max
    """
    # 计算 min 和 max
    agg_df = df.groupby(['patientunitstayid', 'labname']).agg(
        min_value=('labresult_numeric', 'min'),
        max_value=('labresult_numeric', 'max'),
        mean_value=('labresult_numeric', 'mean'),
        count=('labresult_numeric', 'count')
    ).reset_index()
    
    # 转换为宽表
    wide_df = agg_df.pivot(
        index='patientunitstayid',
        columns='labname',
        values=['min_value', 'max_value', 'mean_value', 'count']
    )
    
    # 展平列名
    wide_df.columns = [f"{col[1]}_{col[0]}" for col in wide_df.columns]
    wide_df = wide_df.reset_index()
    
    return wide_df

# 使用示例
if __name__ == "__main__":
    # 示例 1：提取所有患者的血气指标
    blood_gas = extract_first_day_blood_gas()
    print(f"提取了 {len(blood_gas)} 条记录")
    print(blood_gas.head(10))
    
    # 示例 2：转换为宽表
    wide_df = pivot_blood_gas_to_wide(blood_gas)
    print("\n宽表格式:")
    print(wide_df.head(5))
    
    # 示例 3：只提取 pH 和乳酸
    blood_gas_subset = extract_first_day_blood_gas(
        lab_names=['pH', 'lactate']
    )
    print(f"\n提取了 {len(blood_gas_subset)} 条 pH 和乳酸记录")
    
    conn.close()
```

---

## 注意事项

### 1. 使用精确匹配（重要！）

✅ **正确** (精确匹配):
```sql
SELECT l.patientunitstayid, l.labname, l.labresult
FROM lab l
WHERE l.labname = 'pH'  -- 精确匹配
  AND l.labresultoffset >= 0 AND l.labresultoffset < 1440
```

❌ **错误** (模糊匹配):
```sql
SELECT l.patientunitstayid, l.labname, l.labresult
FROM lab l
WHERE l.labname ILIKE '%pH%'  -- 可能匹配到不需要的结果
  AND l.labresultoffset >= 0 AND l.labresultoffset < 1440
```

### 2. 异常值检查（重要！）

基于 eicu-code 和血气分析的常识，查询时务必添加异常值检查：

```sql
SELECT 
    l.patientunitstayid,
    l.labname,
    l.labresult,
    CASE
      WHEN l.labname = 'pH' AND (l.labresult < 6.8 OR l.labresult > 7.8) THEN NULL
      WHEN l.labname = 'pO2' AND (l.labresult < 0 OR l.labresult > 700) THEN NULL
      WHEN l.labname = 'pCO2' AND (l.labresult < 0 OR l.labresult > 150) THEN NULL
      WHEN l.labname = 'O2 saturation' AND (l.labresult < 0 OR l.labresult > 100) THEN NULL
      ELSE l.labresult
    END AS labresult_checked
FROM lab l
WHERE l.labresultoffset >= 0 AND l.labresultoffset < 1440
  AND l.labresult IS NOT NULL
  AND l.labname = 'pH'
```

### 3. 字段名规范

✅ **正确** (小写):
```sql
SELECT l.patientunitstayid, l.labname, l.labresult
FROM lab l
WHERE l.labresultoffset >= 0
```

❌ **错误** (camelCase):
```sql
SELECT l.patientUnitStayID, l.labName, l.labResult  -- 字段不存在！
FROM lab l
WHERE l.labResultOffset >= 0
```

### 4. 数据类型转换

- `labresult` 字段类型为 `numeric(11,4)`，但通常存储为 `varchar`
- 计算时需转换：`CAST(l.labresult AS numeric)` 或 `l.labresult::numeric`
- 转换失败会得到 `NULL`（使用 `NULLIF` 或 `CASE WHEN` 处理异常值）

### 5. 时间范围

- `labresultoffset`: 结果时间（相对于 ICU 入住的分钟数）
- 第一天 = `labresultoffset >= 0 AND labresultoffset < 1440`

### 6. 数据质量

- **缺失值多**: eICU 的血气数据完整性可能较低
- **异常值**: 查询时务必添加合理性检查
- **重复值**: 同一患者同一指标可能有多个记录（不同时间），使用 `MIN`, `MAX`, `AVG` 聚合

---

## 常用查询场景

### 1. 筛选酸中毒患者（pH < 7.35 或乳酸 > 2.0）

```sql
SELECT DISTINCT l.patientunitstayid
FROM lab l
WHERE l.labresultoffset >= 0 AND l.labresultoffset < 1440
  AND (
    (l.labname = 'pH' AND l.labresult::numeric < 7.35)
    OR
    (l.labname = 'lactate' AND l.labresult::numeric > 2.0)
  )
ORDER BY l.patientunitstayid;
```

### 2. 提取低氧血症患者（pO2 < 60 mmHg）

```sql
SELECT DISTINCT l.patientunitstayid
FROM lab l
WHERE l.labresultoffset >= 0 AND l.labresultoffset < 1440
  AND l.labname = 'pO2'
  AND l.labresult::numeric < 60  -- 低氧血症
ORDER BY l.patientunitstayid;
```

---

## 参考链接

- **lab 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/lab.html
- **eICU 数据介绍**: https://eicu-crd.mit.edu/

---

**最后更新**: 2026-05-22  
**更新人**: 悟空（基于 eICU 2.0 官方文档和 eicu-code 逻辑）
