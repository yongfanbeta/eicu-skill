# eICU 2.0 尿量（Urine Output）查询

**参考**: 
- eicu-code 官方实现: `D:\Project\eicu-code\concepts\pivoted\pivoted-uo.sql`
- eICU 2.0 SchemaSpy 文档: https://lcp.mit.edu/eicu-schema-spy/

---

## 目录

1. [为什么使用 intakeoutput 表？](#为什么使用-intakeoutput-表)
2. [识别尿量记录](#识别尿量记录)
3. [SQL 模板](#sql-模板)
4. [Python 模板](#python-模板)
5. [注意事项](#注意事项)

---

## 为什么使用 intakeoutput 表？

eicu-code **官方实现使用 `intakeoutput` 表**提取尿量。

**原因**:
1. **数据完整性更好**: `intakeoutput` 表专门记录入出量（Intake and Output，I&O）
2. **标准化更高**: eicu-code 已经处理了数据清洗和异常值检查
3. **性能更好**: 可以通过 `cellpath` 快速过滤

**重要**: 尿量数据**不在** `nursecharting` 表中！

---

## 识别尿量记录

`intakeoutput` 表使用 `cellpath` 字段来标识不同类型的入出量记录。

**重要**: `cellpath` 是一个长字符串，格式为：
```
flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine
```

### 1. 精确匹配（推荐）

eicu-code 提供了一个很长的 `cellpath` 列表，包含所有可能的尿量记录路径。

**常用 `cellpath` 值**（部分列表，完整列表见 eicu-code 官方实现）：

| cellpath | 说明 |
|----------|------|
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|Urine'` | 尿量（最常用） |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|3 way foley'` | 三腔 Foley 尿管 |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|Actual Urine'` | 实际尿量 |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|BRP (urine)'` | BRP（尿量） |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|condome cath urine'` |  condom 导尿管尿量 |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|diaper urine'` | 尿布尿量 |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|inc of urine'` | 大小便尿量 |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|incontient urine'` | 尿失禁尿量 |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|indwelling foley'` | 留置 Foley 尿管 |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|Straight Catheter-Foley'` | 间歇导尿-Foley |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|Suprapubic Urine Output'` | 耻骨上膀胱造瘘尿量 |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|true urine'` | 真实尿量 |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|Urethal Catheter'` | 尿道导尿管 |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|urinary output 7AM - 7 PM'` | 尿量（7AM-7PM） |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|urine'` | 尿量（小写） |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|URINE'` | 尿量（大写） |
| `'flowsheet\|Flowsheet Cell Labels\|I&O\|Output (ml)\|URINE CATHETER'` | 尿量（导尿管） |
| ... | ... |

⚠️ **注意**: 以上 `cellpath` 值来自 eicu-code 官方实现。**不同站点可能略有不同**，建议先查询确认：

```sql
-- 查看所有与尿量相关的 cellpath（前 100 个）
SELECT DISTINCT cellpath, COUNT(*) AS cnt
FROM intakeoutput
WHERE cellpath ILIKE '%urine%' OR cellpath ILIKE '%foley%' OR cellpath ILIKE '%cath%'
GROUP BY cellpath
ORDER BY cnt DESC
LIMIT 100;
```

### 2. 模糊匹配（备选）

如果精确匹配列表太长，可以使用模糊匹配：

```sql
WHERE(cellpath ILIKE '%urine%' OR cellpath ILIKE '%foley%' OR cellpath ILIKE '%cath%')
```

**但是**，eicu-code **不推荐**模糊匹配，因为可能匹配到非尿量记录（如 "urine culture" 等）。

---

## SQL 模板

### 1. 提取 ICU 第一天的尿量（长格式，基于 eicu-code）

```sql
-- 基于 eicu-code 官方实现：D:\Project\eicu-code\concepts\pivoted\pivoted-uo.sql
-- 提取 ICU 第一天（0-1440 分钟）的尿量（长格式）
WITH uo AS (
    SELECT
        patientunitstayid,
        intakeoutputoffset,
        outputtotal,
        cellvaluenumeric,
        CASE
            -- 精确匹配：所有已知的尿量 cellpath
            WHEN cellpath IN (
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|3 way foley',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|3 Way Foley',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Actual Urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Adjusted total UO NOC end shift',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|BRP (urine)',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|BRP (Urine)',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|condome cath urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|diaper urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|inc of urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|incontient urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Incontient urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|incontient Urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|incontinence of urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Incontinence-urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|incontinence/ voids urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|incontinent of urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|INCONTINENT OF URINE',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Incontinent UOP',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|incontinent urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Incontinent (urine)',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Incontinent Urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|incontinent urine counts',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|incont of urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|incont. of urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|incont. of urine count',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|incont urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|incont. urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Incont. urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Incont. Urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|inc urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|inc. urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Inc. urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Inc Urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|indwelling foley',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Indwelling Foley',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Straight Catheter-Foley',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Straight Catheterization Urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Straight Cath UOP',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|straight cath urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Straight Cath Urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|strait cath Urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Suprapubic Urine Output',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|true urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|True Urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|True Urine out',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|unmeasured urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Unmeasured Urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|unmeasured urine output',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urethal Catheter',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urethral Catheter',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|urinary output 7AM - 7 PM',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|urinary output 7AM-7PM',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|URINE',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|URINE CATHETER',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Intermittent/Straight Cath (mL)',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|straightcath',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|straight cath',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Straight cath',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Straight  cath',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Straight Cath',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Straight Cath''d',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|straight cath daily',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|straight cath ml''s',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|straight cath ml''s',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Straight Cath Q6hrs',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|straight caths',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Straight Cath UOP',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|straight cath urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Straight Cath Urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-straight cath',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine-straight cath',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Straight Cath',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Condom Catheter',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|condom catheter',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|condome cath urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|condom cath',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Condom Cath',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|CONDOM CATHETER OUTPUT',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine via condom catheter',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine-foley',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine- foley',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine- Foley',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine foley catheter',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine, L neph:',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine (measured)',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|urine output',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-external catheter',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-foley',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-Foley',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-FOLEY',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-foley catheter',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-FOLEY CATH',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-foley catheter',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-FOLEY CATHETER',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-Foley Output',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-Fpley',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-Ileoconduit',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-left nephrostomy',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-Left Nephrostomy',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-Left Nephrostomy Tube',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-LEFT PCN TUBE',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-L Nephrostomy',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-L Nephrostomy Tube',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-Nephrostomy',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-right nephrostomy',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-RIGHT Nephrouretero Stent Urine Output',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-R nephrostomy',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-R Nephrostomy',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-R. Nephrostomy',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-R Nephrostomy Tube',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-Rt Nephrectomy',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-stent',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-straight cath',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-suprapubic',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-Texas Cath',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-Urine',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output-Urine Output',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine, R neph:',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine-straight cath',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Straight Cath',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|urine (void)',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine- void',
                'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine, void:'
            ) THEN 1
            -- 模糊匹配：foley 相关
            WHEN cellpath ILIKE 'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|foley%'
             AND LOWER(cellpath) NOT ILIKE '%pacu%'
             AND LOWER(cellpath) NOT ILIKE '%or%'
             AND LOWER(cellpath) NOT ILIKE '%ir%'
            THEN 1
            -- 其他模糊匹配
            WHEN cellpath ILIKE 'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Output%Urinary Catheter%' THEN 1
            WHEN cellpath ILIKE 'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Output%Urethral Catheter%' THEN 1
            WHEN cellpath ILIKE 'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine Output (mL)%' THEN 1
            WHEN cellpath ILIKE 'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Output%External Urethral%' THEN 1
            WHEN cellpath ILIKE 'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urinary Catheter Output%' THEN 1
            ELSE 0
        END AS cellpath_is_uo
    FROM intakeoutput
    WHERE intakeoutputoffset >= 0 AND intakeoutputoffset < 1440  -- 第一天
)
SELECT
    patientunitstayid,
    intakeoutputoffset AS chartoffset,
    MAX(outputtotal) AS outputtotal,
    SUM(cellvaluenumeric) AS urineoutput
FROM uo
WHERE uo.cellpath_is_uo = 1
  AND cellvaluenumeric IS NOT NULL
GROUP BY patientunitstayid, intakeoutputoffset
ORDER BY patientunitstayid, intakeoutputoffset;
```

### 2. 提取 ICU 第一天的尿量（宽表格式）

```sql
-- 提取 ICU 第一天（0-1440 分钟）的尿量（宽表格式，每个患者一行）
WITH uo AS (
    -- 同上，省略 CTE 定义
    SELECT
        patientunitstayid,
        intakeoutputoffset,
        outputtotal,
        cellvaluenumeric,
        CASE
            -- ... 省略 cellpath_is_uo 的逻辑 ...
        END AS cellpath_is_uo
    FROM intakeoutput
    WHERE intakeoutputoffset >= 0 AND intakeoutputoffset < 1440
)
SELECT
    patientunitstayid,
    SUM(cellvaluenumeric) AS urine_output_total,  -- 总尿量（ml）
    MAX(outputtotal) AS output_total_max,          -- 最大输出总量
    COUNT(*) AS num_records,                      -- 记录数
    MIN(intakeoutputoffset) AS first_record_offset,  -- 第一次记录时间
    MAX(intakeoutputoffset) AS last_record_offset   -- 最后一次记录时间
FROM uo
WHERE uo.cellpath_is_uo = 1
  AND cellvaluenumeric IS NOT NULL
GROUP BY patientunitstayid
ORDER BY patientunitstayid;
```

### 3. 计算每小时尿量

```sql
-- 计算每小时尿量（用于 AKI 诊断）
WITH uo AS (
    -- 同上，省略 CTE 定义
    SELECT
        patientunitstayid,
        intakeoutputoffset,
        cellvaluenumeric,
        CASE
            -- ... 省略 cellpath_is_uo 的逻辑 ...
        END AS cellpath_is_uo
    FROM intakeoutput
    WHERE intakeoutputoffset >= 0 AND intakeoutputoffset < 1440
)
SELECT
    patientunitstayid,
    FLOOR(intakeoutputoffset / 60) AS hour_window,  -- 小时窗口
    SUM(cellvaluenumeric) AS urine_output_per_hour   -- 每小时尿量
FROM uo
WHERE uo.cellpath_is_uo = 1
  AND cellvaluenumeric IS NOT NULL
GROUP BY patientunitstayid, FLOOR(intakeoutputoffset / 60)
ORDER BY patientunitstayid, hour_window;
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

def extract_first_day_urine_output(patient_ids=None):
    """
    从 intakeoutput 表提取 ICU 第一天（0-1440 分钟）的尿量
    （基于 eicu-code 官方实现）
    
    Parameters:
    -----------
    patient_ids : list or None
        患者 ID 列表，None 表示所有患者
    
    Returns:
    --------
    df : pandas DataFrame
        长格式尿量数据
    """
    # 构建查询（省略完整的 cellpath 列表，实际使用时应包含）
    query = """
        WITH uo AS (
            SELECT
                patientunitstayid,
                intakeoutputoffset,
                outputtotal,
                cellvaluenumeric,
                CASE
                    WHEN cellpath IN (
                        'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine',
                        'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|3 way foley',
                        -- ... 省略其他 cellpath ...
                    ) THEN 1
                    WHEN cellpath ILIKE 'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|foley%'
                     AND LOWER(cellpath) NOT ILIKE '%pacu%'
                     AND LOWER(cellpath) NOT ILIKE '%or%'
                     AND LOWER(cellpath) NOT ILIKE '%ir%'
                    THEN 1
                    ELSE 0
                END AS cellpath_is_uo
            FROM intakeoutput
            WHERE intakeoutputoffset >= 0 AND intakeoutputoffset < 1440
        )
        SELECT
            patientunitstayid,
            intakeoutputoffset AS chartoffset,
            MAX(outputtotal) AS outputtotal,
            SUM(cellvaluenumeric) AS urineoutput
        FROM uo
        WHERE uo.cellpath_is_uo = 1
          AND cellvaluenumeric IS NOT NULL
        GROUP BY patientunitstayid, intakeoutputoffset
        ORDER BY patientunitstayid, intakeoutputoffset
    """
    
    # 添加患者 ID 过滤
    if patient_ids is not None:
        if len(patient_ids) == 1:
            query = query.replace(
                "WHERE intakeoutputoffset",
                f"WHERE patientunitstayid = {patient_ids[0]} AND intakeoutputoffset"
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
                "FROM intakeoutput",
                "FROM intakeoutput, target_patients tp WHERE patientunitstayid = tp.pid"
            )
    
    # 执行查询
    df = pd.read_sql_query(query, conn)
    
    return df

def aggregate_urine_output_to_wide(df):
    """
    将长格式尿量数据转换为宽表格式
    
    Parameters:
    -----------
    df : pandas DataFrame
        长格式数据（来自 extract_first_day_urine_output）
    
    Returns:
    --------
    wide_df : pandas DataFrame
        宽表格式，每个患者一行
    """
    # 计算统计量
    agg_df = df.groupby('patientunitstayid').agg(
        urine_output_total=('urineoutput', 'sum'),
        output_total_max=('outputtotal', 'max'),
        num_records=('urineoutput', 'count'),
        first_record_offset=('chartoffset', 'min'),
        last_record_offset=('chartoffset', 'max')
    ).reset_index()
    
    return agg_df

# 使用示例
if __name__ == "__main__":
    # 示例 1：提取所有患者的尿量
    uo = extract_first_day_urine_output()
    print(f"提取了 {len(uo)} 条记录")
    print(uo.head(10))
    
    # 示例 2：提取特定患者的尿量
    uo = extract_first_day_urine_output(
        patient_ids=[154347, 112001, 189456]
    )
    
    # 示例 3：转换为宽表
    wide_df = aggregate_urine_output_to_wide(uo)
    print("\n宽表格式:")
    print(wide_df.head(5))
    
    conn.close()
```

---

## 注意事项

### 1. 使用精确匹配（重要！）

✅ **正确** (精确匹配):
```sql
WHERE cellpath IN (
    'flowsheet|Flowsheet Cell Labels|I&O|Output (ml)|Urine',
    -- ... 其他 cellpath ...
)
```

❌ **错误** (模糊匹配):
```sql
WHERE cellpath ILIKE '%urine%'
```
（可能匹配到非尿量记录，如 "urine culture"）

### 2. 异常值检查（重要！）

尿量应该是**正数**，且不应该太大（如 > 2000 ml/h 可能是错误记录）。

```sql
-- 添加异常值检查
WHERE cellvaluenumeric > 0 AND cellvaluenumeric < 2000
```

### 3. 数据类型

- `cellvaluenumeric` 字段类型为 `NUMERIC`
- `outputtotal` 字段类型可能为 `VARCHAR` 或 `NUMERIC`，需要检查

### 4. 性能优化

- `intakeoutput` 表有 **大量记录**（具体数量未查，但应该很大）
- 查询时务必:
  - 使用 `patientunitstayid` 过滤
  - 限定 `intakeoutputoffset` 范围
  - 使用 `cellpath` 过滤（如 `cellpath IN (...)`）
  - 添加 `LIMIT` 子句（测试时）
  - 使用 `EXPLAIN` 分析查询计划

### 5. 时间偏移量

- `intakeoutputoffset`: 记录时间（相对于 ICU 入住的分钟数）
- 第一天 = `intakeoutputoffset >= 0 AND intakeoutputoffset < 1440`

---

## 常用查询场景

### 1. 筛选少尿患者（尿量 < 0.5 ml/kg/h）

```sql
-- 筛选 ICU 第一天少尿的患者（尿量 < 0.5 ml/kg/h）
WITH uo AS (
    -- 同上，省略 CTE 定义
    SELECT
        patientunitstayid,
        SUM(cellvaluenumeric) AS urine_output_total,
        MAX(p.admissionweight) AS admissionweight
    FROM intakeoutput io
    INNER JOIN patient p ON io.patientunitstayid = p.patientunitstayid
    WHERE io.intakeoutputoffset >= 0 AND io.intakeoutputoffset < 1440
      AND io.cellpath_is_uo = 1
      AND io.cellvaluenumeric IS NOT NULL
    GROUP BY io.patientunitstayid
)
SELECT
    patientunitstayid,
    urine_output_total,
    admissionweight,
    urine_output_total / (admissionweight * 24) AS urine_output_per_kg_per_hour
FROM uo
WHERE urine_output_total / (admissionweight * 24) < 0.5  -- 少尿阈值
ORDER BY patientunitstayid;
```

### 2. 计算尿量随时间的变化

```sql
-- 计算尿量随时间的变化（ICU 第一天，每 6 小时）
WITH uo AS (
    -- 同上，省略 CTE 定义
    SELECT
        patientunitstayid,
        intakeoutputoffset,
        cellvaluenumeric
    FROM intakeoutput
    WHERE intakeoutputoffset >= 0 AND intakeoutputoffset < 1440
      AND cellpath_is_uo = 1
      AND cellvaluenumeric IS NOT NULL
)
SELECT
    patientunitstayid,
    FLOOR(intakeoutputoffset / 360) * 360 AS time_window_start,  -- 6 小时窗口
    SUM(cellvaluenumeric) AS urine_output_per_6h
FROM uo
GROUP BY patientunitstayid, FLOOR(intakeoutputoffset / 360)
ORDER BY patientunitstayid, time_window_start;
```

---

## 参考链接

- **intakeoutput 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/intakeoutput.html
- **eicu-code 官方实现**: `D:\Project\eicu-code\concepts\pivoted\pivoted-uo.sql`
- **eICU 数据介绍**: https://eicu-crd.mit.edu/

---

**最后更新**: 2026-05-22  
**更新人**: 悟空（基于 eicu-code 官方实现）
