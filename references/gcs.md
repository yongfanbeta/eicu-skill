# eICU 2.0 GCS（格拉斯哥昏迷评分）查询

**参考**: 
- eicu-code 官方实现: `D:\Project\eicu-code\concepts\pivoted\pivoted-gcs.sql`
- eICU 2.0 SchemaSpy 文档: https://lcp.mit.edu/eicu-schema-spy/

---

## 目录

1. [指标映射](#指标映射)
2. [SQL 模板](#sql-模板)
3. [Python 模板](#python-模板)
4. [注意事项](#注意事项)

---

## 指标映射

GCS 数据来自 `nursecharting` 表，使用 `nursingchartcelltypevlabel` 和 `nursingchartcelltypevalname` 来标识。

**重要**: 提取时务必使用**精确匹配**（`nursingchartcelltypevlabel = 'xxx'` 和 `nursingchartcelltypevalname = 'xxx'`）。

| 指标 | nursingchartcelltypevlabel | nursingchartcelltypevalname | 范围 | 说明 |
|------|---------------------------|----------------------------|------|------|
| GCS 总分 | `'Glasgow coma score'` | `'GCS Total'` | 3-15 | 总分 |
| GCS 总分（另一种来源） | `'Score (Glasgow Coma Scale)'` | `'Value'` | 3-15 | 总分 |
| GCS 运动评分 | `'Glasgow coma score'` | `'Motor'` | 1-6 | 运动反应 |
| GCS 语言评分 | `'Glasgow coma score'` | `'Verbal'` | 1-5 | 语言反应 |
| GCS 睁眼评分 | `'Glasgow coma score'` | `'Eyes'` | 1-4 | 睁眼反应 |

⚠️ **注意**: 以上 `nursingchartcelltypevlabel` 和 `nursingchartcelltypevalname` 值来自 eicu-code 官方实现。**不同站点可能略有不同**，建议先查询确认：

```sql
-- 查看所有与 GCS 相关的 nursingchartcelltypevlabel 和 nursingchartcelltypevalname
SELECT DISTINCT nursingchartcelltypevlabel, nursingchartcelltypevalname, COUNT(*) AS cnt
FROM nursecharting
WHERE nursingchartcelltypevlabel ILIKE '%glasgow%' OR nursingchartcelltypevlabel ILIKE '%gcs%'
GROUP BY nursingchartcelltypevlabel, nursingchartcelltypevalname
ORDER BY cnt DESC;
```

---

## SQL 模板

### 1. 提取第一次 GCS 评分（基于 eicu-code）

```sql
-- 基于 eicu-code 官方实现：D:\Project\eicu-code\concepts\pivoted\pivoted-gcs.sql
-- 提取每个患者的第一次 GCS 评分（最小的 chartoffset）
WITH first_gcs AS (
    SELECT
        nc.patientunitstayid,
        nc.nursingchartoffset,
        nc.gcs,
        nc.gcsmotor,
        nc.gcsverbal,
        nc.gcseyes,
        ROW_NUMBER() OVER (
            PARTITION BY nc.patientunitstayid
            ORDER BY nc.nursingchartoffset
        ) AS rn
    FROM (
        SELECT
            patientunitstayid,
            nursingchartoffset,
            MIN(CASE
                WHEN nursingchartcelltypevlabel = 'Glasgow coma score'
                 AND nursingchartcelltypevalname = 'GCS Total'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
                WHEN nursingchartcelltypevlabel = 'Score (Glasgow Coma Scale)'
                 AND nursingchartcelltypevalname = 'Value'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
                ELSE NULL
            END) AS gcs,
            MIN(CASE
                WHEN nursingchartcelltypevlabel = 'Glasgow coma score'
                 AND nursingchartcelltypevalname = 'Motor'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
                ELSE NULL
            END) AS gcsmotor,
            MIN(CASE
                WHEN nursingchartcelltypevlabel = 'Glasgow coma score'
                 AND nursingchartcelltypevalname = 'Verbal'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
                ELSE NULL
            END) AS gcsverbal,
            MIN(CASE
                WHEN nursingchartcelltypevlabel = 'Glasgow coma score'
                 AND nursingchartcelltypevalname = 'Eyes'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
                ELSE NULL
            END) AS gcseyes
        FROM nursecharting
        WHERE nursingchartcelltypecat IN ('Scores', 'Other Vital Signs and Infusions')
        GROUP BY patientunitstayid, nursingchartoffset
    ) nc
    WHERE nc.gcs IS NOT NULL
       OR nc.gcsmotor IS NOT NULL
       OR nc.gcsverbal IS NOT NULL
       OR nc.gcseyes IS NOT NULL
)
SELECT *
FROM first_gcs
WHERE rn = 1
ORDER BY patientunitstayid;
```

### 2. 提取 ICU 第一天的 GCS 评分（长格式）

```sql
-- 提取 ICU 第一天（0-1440 分钟）的 GCS 评分（长格式）
SELECT
    patientunitstayid,
    nursingchartoffset AS chartoffset,
    'gcs' AS vital_name,
    gcs AS valuenum
FROM (
    SELECT
        patientunitstayid,
        nursingchartoffset,
        MIN(CASE
            WHEN nursingchartcelltypevlabel = 'Glasgow coma score'
                 AND nursingchartcelltypevalname = 'GCS Total'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
            WHEN nursingchartcelltypevlabel = 'Score (Glasgow Coma Scale)'
                 AND nursingchartcelltypevalname = 'Value'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END) AS gcs
    FROM nursecharting
    WHERE nursingchartcelltypecat IN ('Scores', 'Other Vital Signs and Infusions')
      AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
    GROUP BY patientunitstayid, nursingchartoffset
) gcs_data
WHERE gcs IS NOT NULL
  AND gcs >= 3 AND gcs <= 15  -- 异常值检查

UNION ALL

SELECT
    patientunitstayid,
    nursingchartoffset AS chartoffset,
    'gcs_motor' AS vital_name,
    gcsmotor AS valuenum
FROM (
    SELECT
        patientunitstayid,
        nursingchartoffset,
        MIN(CASE
            WHEN nursingchartcelltypevlabel = 'Glasgow coma score'
                 AND nursingchartcelltypevalname = 'Motor'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END) AS gcsmotor
    FROM nursecharting
    WHERE nursingchartcelltypecat IN ('Scores', 'Other Vital Signs and Infusions')
      AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
    GROUP BY patientunitstayid, nursingchartoffset
) gcs_motor_data
WHERE gcsmotor IS NOT NULL
  AND gcsmotor >= 1 AND gcsmotor <= 6  -- 异常值检查

UNION ALL

SELECT
    patientunitstayid,
    nursingchartoffset AS chartoffset,
    'gcs_verbal' AS vital_name,
    gcsverbal AS valuenum
FROM (
    SELECT
        patientunitstayid,
        nursingchartoffset,
        MIN(CASE
            WHEN nursingchartcelltypevlabel = 'Glasgow coma score'
                 AND nursingchartcelltypevalname = 'Verbal'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END) AS gcsverbal
    FROM nursecharting
    WHERE nursingchartcelltypecat IN ('Scores', 'Other Vital Signs and Infusions')
      AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
    GROUP BY patientunitstayid, nursingchartoffset
) gcs_verbal_data
WHERE gcsverbal IS NOT NULL
  AND gcsverbal >= 1 AND gcsverbal <= 5  -- 异常值检查

UNION ALL

SELECT
    patientunitstayid,
    nursingchartoffset AS chartoffset,
    'gcs_eyes' AS vital_name,
    gcseyes AS valuenum
FROM (
    SELECT
        patientunitstayid,
        nursingchartoffset,
        MIN(CASE
            WHEN nursingchartcelltypevlabel = 'Glasgow coma score'
                 AND nursingchartcelltypevalname = 'Eyes'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END) AS gcseyes
    FROM nursecharting
    WHERE nursingchartcelltypecat IN ('Scores', 'Other Vital Signs and Infusions')
      AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
    GROUP BY patientunitstayid, nursingchartoffset
) gcs_eyes_data
WHERE gcseyes IS NOT NULL
  AND gcseyes >= 1 AND gcseyes <= 4  -- 异常值检查

ORDER BY patientunitstayid, chartoffset, vital_name;
```

### 3. 提取 ICU 第一天的 GCS 评分（宽表格式）

```sql
-- 提取 ICU 第一天（0-1440 分钟）的 GCS 评分（宽表格式，每个患者一行）
WITH gcs_data AS (
    SELECT
        patientunitstayid,
        nursingchartoffset,
        MIN(CASE
            WHEN nursingchartcelltypevlabel = 'Glasgow coma score'
                 AND nursingchartcelltypevalname = 'GCS Total'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
            WHEN nursingchartcelltypevlabel = 'Score (Glasgow Coma Scale)'
                 AND nursingchartcelltypevalname = 'Value'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END) AS gcs,
        MIN(CASE
            WHEN nursingchartcelltypevlabel = 'Glasgow coma score'
                 AND nursingchartcelltypevalname = 'Motor'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END) AS gcsmotor,
        MIN(CASE
            WHEN nursingchartcelltypevlabel = 'Glasgow coma score'
                 AND nursingchartcelltypevalname = 'Verbal'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END) AS gcsverbal,
        MIN(CASE
            WHEN nursingchartcelltypevlabel = 'Glasgow coma score'
                 AND nursingchartcelltypevalname = 'Eyes'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END) AS gcseyes
    FROM nursecharting
    WHERE nursingchartcelltypecat IN ('Scores', 'Other Vital Signs and Infusions')
      AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
    GROUP BY patientunitstayid, nursingchartoffset
)
SELECT
    patientunitstayid,
    AVG(CASE WHEN gcs >= 3 AND gcs <= 15 THEN gcs ELSE NULL END) AS gcs_mean,
    MIN(CASE WHEN gcs >= 3 AND gcs <= 15 THEN gcs ELSE NULL END) AS gcs_min,
    MAX(CASE WHEN gcs >= 3 AND gcs <= 15 THEN gcs ELSE NULL END) AS gcs_max,
    AVG(CASE WHEN gcsmotor >= 1 AND gcsmotor <= 6 THEN gcsmotor ELSE NULL END) AS gcsmotor_mean,
    AVG(CASE WHEN gcsverbal >= 1 AND gcsverbal <= 5 THEN gcsverbal ELSE NULL END) AS gcsverbal_mean,
    AVG(CASE WHEN gcseyes >= 1 AND gcseyes <= 4 THEN gcseyes ELSE NULL END) AS gcseyes_mean
FROM gcs_data
WHERE gcs IS NOT NULL
   OR gcsmotor IS NOT NULL
   OR gcsverbal IS NOT NULL
   OR gcseyes IS NOT NULL
GROUP BY patientunitstayid
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

# GCS 的 nursingchartcelltypevlabel 和 nursingchartcelltypevalname 的准确值（基于 eicu-code）
GCS_MAP = {
    'gcs': {
        'label': ['Glasgow coma score', 'Score (Glasgow Coma Scale)'],
        'valname': ['GCS Total', 'Value'],
        'min_value': 3,
        'max_value': 15
    },
    'gcsmotor': {
        'label': ['Glasgow coma score'],
        'valname': ['Motor'],
        'min_value': 1,
        'max_value': 6
    },
    'gcsverbal': {
        'label': ['Glasgow coma score'],
        'valname': ['Verbal'],
        'min_value': 1,
        'max_value': 5
    },
    'gcseyes': {
        'label': ['Glasgow coma score'],
        'valname': ['Eyes'],
        'min_value': 1,
        'max_value': 4
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

def extract_first_day_gcs(patient_ids=None, gcs_components=None):
    """
    从 nursecharting 表提取 ICU 第一天（0-1440 分钟）的 GCS 评分
    （基于 eicu-code 官方实现）
    
    Parameters:
    -----------
    patient_ids : list or None
        患者 ID 列表，None 表示所有患者
    gcs_components : list or None
        要提取的 GCS 分量列表，None 表示全部
        可选值: 'gcs', 'gcsmotor', 'gcsverbal', 'gcseyes'
    
    Returns:
    --------
    df : pandas DataFrame
        长格式 GCS 数据
    """
    if gcs_components is None:
        gcs_components = list(GCS_MAP.keys())
    
    # 构建 CASE 子句
    case_clauses = []
    for component in gcs_components:
        if component not in GCS_MAP:
            continue
        
        config = GCS_MAP[component]
        case_clause = f"""
        MIN(CASE
        """
        
        for i in range(len(config['label'])):
            if i > 0:
                case_clause += "            WHEN "
            else:
                case_clause += "            WHEN "
            
            case_clause += f"nursingchartcelltypevlabel = '{config['label'][i]}' "
            case_clause += f"AND nursingchartcelltypevalname = '{config['valname'][i]}' "
            case_clause += "AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$' "
            case_clause += "AND nursingchartvalue NOT IN ('-', '.') "
            case_clause += f"               THEN CAST(nursingchartvalue AS NUMERIC) "
        
        case_clause += f"           ELSE NULL END) AS {component}"
        case_clauses.append(case_clause)
    
    if not case_clauses:
        raise ValueError("没有有效的 GCS 分量配置")
    
    # 构建查询
    query = f"""
        SELECT
            patientunitstayid,
            nursingchartoffset,
            {','.join(case_clauses)}
        FROM nursecharting
        WHERE nursingchartcelltypecat IN ('Scores', 'Other Vital Signs and Infusions')
          AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
        GROUP BY patientunitstayid, nursingchartoffset
        ORDER BY patientunitstayid, nursingchartoffset
    """
    
    # 添加患者 ID 过滤
    if patient_ids is not None:
        if len(patient_ids) == 1:
            query = query.replace(
                "WHERE nursingchartcelltypecat",
                f"WHERE patientunitstayid = {patient_ids[0]} AND nursingchartcelltypecat"
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
                "FROM nursecharting",
                "FROM nursecharting, target_patients tp WHERE patientunitstayid = tp.pid"
            )
    
    # 执行查询
    df = pd.read_sql_query(query, conn)
    
    # 转换 GCS 分量为数值（如果可能）
    for component in gcs_components:
        if component in df.columns:
            df[component] = pd.to_numeric(df[component], errors='coerce')
            # 异常值检查
            if component in GCS_MAP:
                min_val = GCS_MAP[component]['min_value']
                max_val = GCS_MAP[component]['max_value']
                df[component] = df[component].apply(
                    lambda x: x if (pd.notnull(x) and min_val <= x <= max_val) else np.nan
                )
    
    return df

def aggregate_gcs_to_wide(df):
    """
    将长格式 GCS 数据转换为宽表格式
    
    Parameters:
    -----------
    df : pandas DataFrame
        长格式数据（来自 extract_first_day_gcs）
    
    Returns:
    --------
    wide_df : pandas DataFrame
        宽表格式，每个患者一行
    """
    # 计算统计量
    agg_df = df.groupby('patientunitstayid').agg(
        {col: ['mean', 'min', 'max', 'count'] for col in df.columns if col not in ['patientunitstayid', 'nursingchartoffset']}
    ).reset_index()
    
    # 展平列名
    agg_df.columns = ['_'.join(col).strip() if col[1] else col[0] for col in agg_df.columns.values]
    
    return agg_df

# 使用示例
if __name__ == "__main__":
    # 示例 1：提取所有患者的 GCS 总分
    gcs = extract_first_day_gcs(
        gcs_components=['gcs']
    )
    print(f"提取了 {len(gcs)} 条记录")
    print(gcs.head(10))
    
    # 示例 2：提取特定患者的所有 GCS 分量
    gcs = extract_first_day_gcs(
        patient_ids=[154347, 112001, 189456]
    )
    
    # 示例 3：转换为宽表
    wide_df = aggregate_gcs_to_wide(gcs)
    print("\n宽表格式:")
    print(wide_df.head(5))
    
    conn.close()
```

---

## 注意事项

### 1. 使用精确匹配（重要！）

✅ **正确** (精确匹配):
```sql
WHERE nc.nursingchartcelltypevlabel = 'Glasgow coma score'
  AND nc.nursingchartcelltypevalname = 'GCS Total'
```

❌ **错误** (模糊匹配):
```sql
WHERE nc.nursingchartcelltypevlabel ILIKE '%glasgow%'
```

### 2. 异常值检查（重要！）

基于 eicu-code 官方实现，查询时务必添加异常值检查：

```sql
-- GCS 总分范围：3-15
AND gcs >= 3 AND gcs <= 15

-- GCS 运动评分范围：1-6
AND gcsmotor >= 1 AND gcsmotor <= 6

-- GCS 语言评分范围：1-5
AND gcsverbal >= 1 AND gcsverbal <= 5

-- GCS 睁眼评分范围：1-4
AND gcseyes >= 1 AND gcseyes <= 4
```

### 3. 数据类型转换

- `nursingchartvalue` 字段类型为 `VARCHAR`
- 计算时需转换：`CAST(nc.nursingchartvalue AS NUMERIC)` 或 `nc.nursingchartvalue::NUMERIC`
- **务必先检查是否为有效数字**：`nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$')`

### 4. 性能优化

- `nursecharting` 表有 **1.51 亿条记录**
- 查询时务必:
  - 使用 `patientunitstayid` 过滤
  - 限定 `nursingchartoffset` 范围
  - 使用 `nursingchartcelltypecat` 过滤（如 `'Scores'`）
  - 添加 `LIMIT` 子句（测试时）
  - 使用 `EXPLAIN` 分析查询计划

### 5. 时间偏移量

- `nursingchartoffset`: 记录时间（相对于 ICU 入住的分钟数）
- 第一天 = `nursingchartoffset >= 0 AND nursingchartoffset < 1440`

---

## 常用查询场景

### 1. 筛选重度昏迷患者（GCS ≤ 8）

```sql
-- 筛选 ICU 第一天 GCS ≤ 8 的患者（重度昏迷）
SELECT DISTINCT patientunitstayid
FROM (
    SELECT
        patientunitstayid,
        MIN(CASE
            WHEN nursingchartcelltypevlabel = 'Glasgow coma score'
                 AND nursingchartcelltypevalname = 'GCS Total'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
            WHEN nursingchartcelltypevlabel = 'Score (Glasgow Coma Scale)'
                 AND nursingchartcelltypevalname = 'Value'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END) AS gcs
    FROM nursecharting
    WHERE nursingchartcelltypecat IN ('Scores', 'Other Vital Signs and Infusions')
      AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
    GROUP BY patientunitstayid, nursingchartoffset
) gcs_data
WHERE gcs IS NOT NULL
  AND gcs <= 8  -- 重度昏迷
ORDER BY patientunitstayid;
```

### 2. 计算 GCS 评分随时间的变化

```sql
-- 计算 GCS 评分随时间的变化（ICU 第一天，每 6 小时）
WITH gcs_data AS (
    SELECT
        patientunitstayid,
        nursingchartoffset,
        MIN(CASE
            WHEN nursingchartcelltypevlabel = 'Glasgow coma score'
                 AND nursingchartcelltypevalname = 'GCS Total'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
            WHEN nursingchartcelltypevlabel = 'Score (Glasgow Coma Scale)'
                 AND nursingchartcelltypevalname = 'Value'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END) AS gcs
    FROM nursecharting
    WHERE nursingchartcelltypecat IN ('Scores', 'Other Vital Signs and Infusions')
      AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
    GROUP BY patientunitstayid, nursingchartoffset
)
SELECT
    patientunitstayid,
    FLOOR(nursingchartoffset / 360) * 360 AS time_window_start,  -- 6 小时窗口
    AVG(CASE WHEN gcs >= 3 AND gcs <= 15 THEN gcs ELSE NULL END) AS gcs_mean,
    COUNT(*) AS num_records
FROM gcs_data
WHERE gcs IS NOT NULL
GROUP BY patientunitstayid, FLOOR(nursingchartoffset / 360)
ORDER BY patientunitstayid, time_window_start;
```

---

## 参考链接

- **nursecharting 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/nursecharting.html
- **eicu-code 官方实现**: `D:\Project\eicu-code\concepts\pivoted\pivoted-gcs.sql`
- **eICU 数据介绍**: https://eicu-crd.mit.edu/

---

**最后更新**: 2026-05-22  
**更新人**: 悟空（基于 eicu-code 官方实现）
