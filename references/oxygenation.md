# eICU 2.0 氧合（Oxygenation）查询

**参考**: 
- eicu-code 官方实现: `D:\Project\eicu-code\concepts\pivoted\pivoted-o2.sql`
- eICU 2.0 SchemaSpy 文档: https://lcp.mit.edu/eicu-schema-spy/

---

## 目录

1. [指标映射](#指标映射)
2. [SQL 模板](#sql-模板)
3. [Python 模板](#python-模板)
4. [注意事项](#注意事项)

---

## 指标映射

氧合数据来自 `nursecharting` 表，使用 `nursingchartcelltypevlabel` 和 `nursingchartcelltypevalname` 来标识。

**重要**: 提取时务必使用**精确匹配**（`nursingchartcelltypevlabel = 'xxx'` 和 `nursingchartcelltypevalname = 'xxx'`）。

| 指标 | nursingchartcelltypevlabel | nursingchartcelltypevalname | 范围 | 说明 |
|------|---------------------------|----------------------------|------|------|
| 氧流量 (O2 Flow) | `'O2 L/%'` | `'O2 L/%'` | 0-100 L/min | 氧流量 |
| 给氧装置 (O2 Device) | `'O2 Admin Device'` | `'O2 Admin Device'` | 文本 | 给氧装置（如 "Nasal Cannula"、"Mask" 等） |
| 呼气末 CO2 (End Tidal CO2) | `'End Tidal CO2'` | `'End Tidal CO2'` | 0-1000 mmHg | 呼气末 CO2 |

⚠️ **注意**: 以上 `nursingchartcelltypevlabel` 和 `nursingchartcelltypevalname` 值来自 eicu-code 官方实现。**不同站点可能略有不同**，建议先查询确认：

```sql
-- 查看所有与氧合相关的 nursingchartcelltypevlabel 和 nursingchartcelltypevalname
SELECT DISTINCT nursingchartcelltypevlabel, nursingchartcelltypevalname, COUNT(*) AS cnt
FROM nursecharting
WHERE nursingchartcelltypevlabel ILIKE '%o2%' OR nursingchartcelltypevlabel ILIKE '%oxygen%'
GROUP BY nursingchartcelltypevlabel, nursingchartcelltypevalname
ORDER BY cnt DESC;
```

---

## SQL 模板

### 1. 提取 ICU 第一天的氧合数据（长格式，基于 eicu-code）

```sql
-- 基于 eicu-code 官方实现：D:\Project\eicu-code\concepts\pivoted\pivoted-o2.sql
-- 提取 ICU 第一天（0-1440 分钟）的氧合数据（长格式）
WITH nc AS (
    SELECT
        patientunitstayid,
        nursingchartoffset,
        nursingchartentryoffset,
        CASE
            WHEN nursingchartcelltypevlabel = 'O2 L/%'
             AND nursingchartcelltypevalname = 'O2 L/%'
             -- 验证是否为数字
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS o2_flow,
        CASE
            WHEN nursingchartcelltypevlabel = 'O2 Admin Device'
             AND nursingchartcelltypevalname = 'O2 Admin Device'
                THEN nursingchartvalue
            ELSE NULL
        END AS o2_device,
        CASE
            WHEN nursingchartcelltypevlabel = 'End Tidal CO2'
             AND nursingchartcelltypevalname = 'End Tidal CO2'
             -- 验证是否为数字
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS etco2
    FROM nursecharting
    -- 只查看生命体征类别
    WHERE nursingchartcelltypecat = 'Vital Signs'
      AND nursingchartoffset >= 0 AND nursingchartoffset < 1440  -- 第一天
)
SELECT
    patientunitstayid,
    nursingchartoffset AS chartoffset,
    nursingchartentryoffset AS entryoffset,
    AVG(CASE WHEN o2_flow >= 0 AND o2_flow <= 100 THEN o2_flow ELSE NULL END) AS o2_flow,
    MAX(o2_device) AS o2_device,  -- 取最常见的值
    AVG(CASE WHEN etco2 >= 0 AND etco2 <= 1000 THEN etco2 ELSE NULL END) AS etco2
FROM nc
WHERE o2_flow IS NOT NULL
   OR o2_device IS NOT NULL
   OR etco2 IS NOT NULL
GROUP BY patientunitstayid, nursingchartoffset, nursingchartentryoffset
ORDER BY patientunitstayid, nursingchartoffset, nursingchartentryoffset;
```

### 2. 提取 ICU 第一天的氧合数据（宽表格式）

```sql
-- 提取 ICU 第一天（0-1440 分钟）的氧合数据（宽表格式，每个患者一行）
WITH nc AS (
    -- 同上，省略 CTE 定义
    SELECT
        patientunitstayid,
        nursingchartoffset,
        nursingchartentryoffset,
        CASE
            WHEN nursingchartcelltypevlabel = 'O2 L/%'
             AND nursingchartcelltypevalname = 'O2 L/%'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS o2_flow,
        CASE
            WHEN nursingchartcelltypevlabel = 'O2 Admin Device'
             AND nursingchartcelltypevalname = 'O2 Admin Device'
                THEN nursingchartvalue
            ELSE NULL
        END AS o2_device,
        CASE
            WHEN nursingchartcelltypevlabel = 'End Tidal CO2'
             AND nursingchartcelltypevalname = 'End Tidal CO2'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS etco2
    FROM nursecharting
    WHERE nursingchartcelltypecat = 'Vital Signs'
      AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
)
SELECT
    patientunitstayid,
    AVG(CASE WHEN o2_flow >= 0 AND o2_flow <= 100 THEN o2_flow ELSE NULL END) AS o2_flow_mean,
    MAX(o2_device) AS o2_device,  -- 取最常见的值
    AVG(CASE WHEN etco2 >= 0 AND etco2 <= 1000 THEN etco2 ELSE NULL END) AS etco2_mean
FROM nc
WHERE o2_flow IS NOT NULL
   OR o2_device IS NOT NULL
   OR etco2 IS NOT NULL
GROUP BY patientunitstayid
ORDER BY patientunitstayid;
```

### 3. 提取第一次氧合记录

```sql
-- 提取每个患者的第一次氧合记录（最小的 nursingchartoffset）
WITH first_o2 AS (
    SELECT
        nc.patientunitstayid,
        nc.nursingchartoffset,
        nc.o2_flow,
        nc.o2_device,
        nc.etco2,
        ROW_NUMBER() OVER (
            PARTITION BY nc.patientunitstayid
            ORDER BY nc.nursingchartoffset
        ) AS rn
    FROM (
        SELECT
            patientunitstayid,
            nursingchartoffset,
            CASE
                WHEN nursingchartcelltypevlabel = 'O2 L/%'
                 AND nursingchartcelltypevalname = 'O2 L/%'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
                ELSE NULL
            END AS o2_flow,
            CASE
                WHEN nursingchartcelltypevlabel = 'O2 Admin Device'
                 AND nursingchartcelltypevalname = 'O2 Admin Device'
                    THEN nursingchartvalue
                ELSE NULL
            END AS o2_device,
            CASE
                WHEN nursingchartcelltypevlabel = 'End Tidal CO2'
                 AND nursingchartcelltypevalname = 'End Tidal CO2'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
                ELSE NULL
            END AS etco2
        FROM nursecharting
        WHERE nursingchartcelltypecat = 'Vital Signs'
          AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
    ) nc
)
SELECT *
FROM first_o2
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

# 氧合数据的 nursingchartcelltypevlabel 和 nursingchartcelltypevalname 的准确值（基于 eicu-code）
OXYGENATION_MAP = {
    'o2_flow': {
        'label': 'O2 L/%',
        'valname': 'O2 L/%',
        'min_value': 0,
        'max_value': 100
    },
    'o2_device': {
        'label': 'O2 Admin Device',
        'valname': 'O2 Admin Device',
        'min_value': None,
        'max_value': None
    },
    'etco2': {
        'label': 'End Tidal CO2',
        'valname': 'End Tidal CO2',
        'min_value': 0,
        'max_value': 1000
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

def extract_first_day_oxygenation(patient_ids=None, oxygenation_components=None):
    """
    从 nursecharting 表提取 ICU 第一天（0-1440 分钟）的氧合数据
    （基于 eicu-code 官方实现）
    
    Parameters:
    -----------
    patient_ids : list or None
        患者 ID 列表，None 表示所有患者
    oxygenation_components : list or None
        要提取的氧合分量列表，None 表示全部
        可选值: 'o2_flow', 'o2_device', 'etco2'
    
    Returns:
    --------
    df : pandas DataFrame
        长格式氧合数据
    """
    if oxygenation_components is None:
        oxygenation_components = list(OXYGENATION_MAP.keys())
    
    # 构建 CASE 子句
    case_clauses = []
    for component in oxygenation_components:
        if component not in OXYGENATION_MAP:
            continue
        
        config = OXYGENATION_MAP[component]
        if config['min_value'] is not None and config['max_value'] is not None:
            # 数值型字段
            case_clause = f"""
            CASE
                WHEN nursingchartcelltypevlabel = '{config['label']}'
                 AND nursingchartcelltypevalname = '{config['valname']}'
                 AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
                 AND nursingchartvalue NOT IN ('-', '.')
                    THEN CAST(nursingchartvalue AS NUMERIC)
                ELSE NULL
            END AS {component}
            """
        else:
            # 文本型字段
            case_clause = f"""
            CASE
                WHEN nursingchartcelltypevlabel = '{config['label']}'
                 AND nursingchartcelltypevalname = '{config['valname']}'
                    THEN nursingchartvalue
                ELSE NULL
            END AS {component}
            """
        case_clauses.append(case_clause)
    
    if not case_clauses:
        raise ValueError("没有有效的氧合分量配置")
    
    # 构建查询
    query = f"""
        SELECT
            patientunitstayid,
            nursingchartoffset,
            nursingchartentryoffset,
            {','.join(case_clauses)}
        FROM nursecharting
        WHERE nursingchartcelltypecat = 'Vital Signs'
          AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
        ORDER BY patientunitstayid, nursingchartoffset, nursingchartentryoffset
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
    
    # 转换氧合分量为数值（如果可能）
    for component in oxygenation_components:
        if component in df.columns and component != 'o2_device':
            df[component] = pd.to_numeric(df[component], errors='coerce')
            # 异常值检查
            if component in OXYGENATION_MAP:
                min_val = OXYGENATION_MAP[component]['min_value']
                max_val = OXYGENATION_MAP[component]['max_value']
                if min_val is not None and max_val is not None:
                    df[component] = df[component].apply(
                        lambda x: x if (pd.notnull(x) and min_val <= x <= max_val) else np.nan
                    )
    
    return df

def aggregate_oxygenation_to_wide(df):
    """
    将长格式氧合数据转换为宽表格式
    
    Parameters:
    -----------
    df : pandas DataFrame
        长格式数据（来自 extract_first_day_oxygenation）
    
    Returns:
    --------
    wide_df : pandas DataFrame
        宽表格式，每个患者一行
    """
    # 计算统计量
    agg_df = df.groupby('patientunitstayid').agg(
        {col: ['mean', 'min', 'max', 'count'] for col in df.columns if col not in ['patientunitstayid', 'nursingchartoffset', 'nursingchartentryoffset']}
    ).reset_index()
    
    # 展平列名
    agg_df.columns = ['_'.join(col).strip() if col[1] else col[0] for col in agg_df.columns.values]
    
    return agg_df

# 使用示例
if __name__ == "__main__":
    # 示例 1：提取所有患者的氧流量
    o2 = extract_first_day_oxygenation(
        oxygenation_components=['o2_flow']
    )
    print(f"提取了 {len(o2)} 条记录")
    print(o2.head(10))
    
    # 示例 2：提取特定患者的所有氧合数据
    o2 = extract_first_day_oxygenation(
        patient_ids=[154347, 112001, 189456]
    )
    
    # 示例 3：转换为宽表
    wide_df = aggregate_oxygenation_to_wide(o2)
    print("\n宽表格式:")
    print(wide_df.head(5))
    
    conn.close()
```

---

## 注意事项

### 1. 使用精确匹配（重要！）

✅ **正确** (精确匹配):
```sql
WHERE nc.nursingchartcelltypevlabel = 'O2 L/%'
  AND nc.nursingchartcelltypevalname = 'O2 L/%'
```

❌ **错误** (模糊匹配):
```sql
WHERE nc.nursingchartcelltypevlabel ILIKE '%o2%'
```

### 2. 异常值检查（重要！）

基于 eicu-code 官方实现，查询时务必添加异常值检查：

```sql
-- 氧流量范围：0-100 L/min
AND o2_flow >= 0 AND o2_flow <= 100

-- 呼气末 CO2 范围：0-1000 mmHg
AND etco2 >= 0 AND etco2 <= 1000
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

## 常用查询场景

### 1. 筛选高流量氧疗患者（O2 流量 > 6 L/min）

```sql
-- 筛选 ICU 第一天高流量氧疗的患者（O2 流量 > 6 L/min）
SELECT DISTINCT patientunitstayid
FROM (
    SELECT
        patientunitstayid,
        CASE
            WHEN nursingchartcelltypevlabel = 'O2 L/%'
             AND nursingchartcelltypevalname = 'O2 L/%'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS o2_flow
    FROM nursecharting
    WHERE nursingchartcelltypecat = 'Vital Signs'
      AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
) o2_data
WHERE o2_flow IS NOT NULL
  AND o2_flow > 6  -- 高流量氧疗
ORDER BY patientunitstayid;
```

### 2. 计算氧合数据随时间的变化

```sql
-- 计算氧合数据随时间的变化（ICU 第一天，每 6 小时）
WITH o2_data AS (
    SELECT
        patientunitstayid,
        nursingchartoffset,
        CASE
            WHEN nursingchartcelltypevlabel = 'O2 L/%'
             AND nursingchartcelltypevalname = 'O2 L/%'
             AND nursingchartvalue ~ '^[-]?[0-9]+[.]?[0-9]*$'
             AND nursingchartvalue NOT IN ('-', '.')
                THEN CAST(nursingchartvalue AS NUMERIC)
            ELSE NULL
        END AS o2_flow
    FROM nursecharting
    WHERE nursingchartcelltypecat = 'Vital Signs'
      AND nursingchartoffset >= 0 AND nursingchartoffset < 1440
)
SELECT
    patientunitstayid,
    FLOOR(nursingchartoffset / 360) * 360 AS time_window_start,  -- 6 小时窗口
    AVG(CASE WHEN o2_flow >= 0 AND o2_flow <= 100 THEN o2_flow ELSE NULL END) AS o2_flow_mean,
    COUNT(*) AS num_records
FROM o2_data
WHERE o2_flow IS NOT NULL
GROUP BY patientunitstayid, FLOOR(nursingchartoffset / 360)
ORDER BY patientunitstayid, time_window_start;
```

---

## 参考链接

- **nursecharting 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/nursecharting.html
- **eicu-code 官方实现**: `D:\Project\eicu-code\concepts\pivoted\pivoted-o2.sql`
- **eICU 数据介绍**: https://eicu-crd.mit.edu/

---

**最后更新**: 2026-05-22  
**更新人**: 悟空（基于 eicu-code 官方实现）
