# eICU 2.0 升压药（Vasopressors）查询

**参考**: 
- eicu-code 官方实现: `D:\Project\eicu-code\concepts\pivoted\pivoted-treatment-vasopressor.sql`
- eICU 2.0 SchemaSpy 文档: https://lcp.mit.edu/eicu-schema-spy/

---

## 目录

1. [为什么使用 treatment 表？](#为什么使用-treatment-表)
2. [识别升压药治疗](#识别升压药治疗)
3. [SQL 模板](#sql-模板)
4. [Python 模板](#python-模板)
5. [注意事项](#注意事项)

---

## 为什么使用 treatment 表？

eicu-code **官方实现使用 `treatment` 表**提取升压药治疗。

**原因**:
1. **数据完整性更好**: `treatment` 表专门记录治疗（包括升压药）
2. **标准化更高**: eicu-code 已经处理了数据清洗和异常值检查
3. **性能更好**: 可以通过 `treatmentstring` 快速过滤

**重要**: 升压药数据**不在** `medication` 表中！

---

## 识别升压药治疗

`treatment` 表使用 `treatmentstring` 字段来标识不同类型的治疗。

**重要**: `treatmentstring` 是一个长字符串，格式为：
```
cardiovascular|shock|vasopressors|phenylephrine (Neosynephrine)
```

### 1. 精确匹配（推荐）

eicu-code 提供了一个很长的 `treatmentstring` 列表，包含所有可能的升压药治疗。

**常用 `treatmentstring` 值**（部分列表，完整列表见 eicu-code 官方实现）：

| treatmentstring | 说明 | 记录数 |
|----------|------|---------|
| `'cardiovascular\|shock\|vasopressors\|vasopressin'` | 血管加压素 | 11,082 |
| `'cardiovascular\|shock\|vasopressors\|phenylephrine (Neosynephrine)'` | 苯肾上腺素（新福林） | 13,189 |
| `'cardiovascular\|shock\|vasopressors\|norepinephrine > 0.1 micrograms/kg/min'` | 去甲肾上腺素（高剂量） | 24,174 |
| `'cardiovascular\|shock\|vasopressors\|norepinephrine <= 0.1 micrograms/kg/min'` | 去甲肾上腺素（低剂量） | 17,467 |
| `'cardiovascular\|shock\|vasopressors\|epinephrine > 0.1 micrograms/kg/min'` | 肾上腺素（高剂量） | 2,410 |
| `'cardiovascular\|shock\|vasopressors\|epinephrine <= 0.1 micrograms/kg/min'` | 肾上腺素（低剂量） | 2,384 |
| `'cardiovascular\|shock\|vasopressors\|dopamine  5-15 micrograms/kg/min'` | 多巴胺（中剂量） | 4,822 |
| `'cardiovascular\|shock\|vasopressors\|dopamine >15 micrograms/kg/min'` | 多巴胺（高剂量） | 1,102 |
| `'surgery\|cardiac therapies\|vasopressors\|vasopressin'` | 血管加压素（心脏术后） | 356 |
| `'surgery\|cardiac therapies\|vasopressors\|phenylephrine (Neosynephrine)'` | 苯肾上腺素（心脏术后） | 1,000 |
| ... | ... | ... |

⚠️ **注意**: 以上 `treatmentstring` 值来自 eicu-code 官方实现。**不同站点可能略有不同**，建议先查询确认：

```sql
-- 查看所有与升压药相关的 treatmentstring（前 100 个）
SELECT DISTINCT treatmentstring, COUNT(*) AS cnt
FROM treatment
WHERE treatmentstring ILIKE '%vasopressor%' OR treatmentstring ILIKE '%norepinephrine%'
GROUP BY treatmentstring
ORDER BY cnt DESC
LIMIT 100;
```

### 2. 升压药类别

根据 eicu-code，升压药分为以下几类：

| 类别 | 药物 | 剂量分类 |
|------|------|----------|
| **血管加压素** | Vasopressin | 无剂量分类 |
| **苯肾上腺素** | Phenylephrine (Neosynephrine) | 无剂量分类 |
| **去甲肾上腺素** | Norepinephrine | 低剂量（≤0.1 mcg/kg/min）、高剂量（>0.1 mcg/kg/min） |
| **肾上腺素** | Epinephrine | 低剂量（≤0.1 mcg/kg/min）、高剂量（>0.1 mcg/kg/min） |
| **多巴胺** | Dopamine | 低剂量（≤5 mcg/kg/min）、中剂量（5-15 mcg/kg/min）、高剂量（>15 mcg/kg/min） |

---

## SQL 模板

### 1. 提取 ICU 第一天的升压药治疗（长格式，基于 eicu-code）

```sql
-- 基于 eicu-code 官方实现：D:\Project\eicu-code\concepts\pivoted\pivoted-treatment-vasopressor.sql
-- 提取 ICU 第一天（0-1440 分钟）的升压药治疗（长格式）
WITH tr AS (
    SELECT
        patientunitstayid,
        treatmentoffset AS chartoffset,
        MAX(CASE 
            WHEN treatmentstring IN (
                -- 血管加压素
                'toxicology|drug overdose|vasopressors|vasopressin',
                'surgery|cardiac therapies|vasopressors|vasopressin',
                'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|vasopressin',
                'gastrointestinal|medications|hormonal therapy (for varices)|vasopressin',
                'cardiovascular|shock|vasopressors|vasopressin',
                -- 苯肾上腺素
                'toxicology|drug overdose|vasopressors|phenylephrine (Neosynephrine)',
                'surgery|cardiac therapies|vasopressors|phenylephrine (Neosynephrine)',
                'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|phenylephrine (Neosynephrine)',
                'cardiovascular|shock|vasopressors|phenylephrine (Neosynephrine)',
                -- 去甲肾上腺素（高剂量）
                'toxicology|drug overdose|vasopressors|norepinephrine > 0.1 micrograms/kg/min',
                'surgery|cardiac therapies|vasopressors|norepinephrine > 0.1 micrograms/kg/min',
                'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|norepinephrine > 0.1 micrograms/kg/min',
                'cardiovascular|shock|vasopressors|norepinephrine > 0.1 micrograms/kg/min',
                'cardiovascular|ventricular dysfunction|inotropic agent|norepinephrine > 0.1 micrograms/kg/min',
                'cardiovascular|shock|inotropic agent|norepinephrine > 0.1 micrograms/kg/min',
                'cardiovascular|myocardial ischemia / infarction|inotropic agent|norepinephrine > 0.1 micrograms/kg/min',
                -- 去甲肾上腺素（低剂量）
                'toxicology|drug overdose|vasopressors|norepinephrine <= 0.1 micrograms/kg/min',
                'surgery|cardiac therapies|vasopressors|norepinephrine <= 0.1 micrograms/kg/min',
                'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|norepinephrine <= 0.1 micrograms/kg/min',
                'cardiovascular|shock|vasopressors|norepinephrine <= 0.1 micrograms/kg/min',
                'cardiovascular|ventricular dysfunction|inotropic agent|norepinephrine <= 0.1 micrograms/kg/min',
                'cardiovascular|shock|inotropic agent|norepinephrine <= 0.1 micrograms/kg/min',
                'cardiovascular|myocardial ischemia / infarction|inotropic agent|norepinephrine <= 0.1 micrograms/kg/min',
                -- 肾上腺素（高剂量）
                'toxicology|drug overdose|vasopressors|epinephrine > 0.1 micrograms/kg/min',
                'surgery|cardiac therapies|vasopressors|epinephrine > 0.1 micrograms/kg/min',
                'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|epinephrine > 0.1 micrograms/kg/min',
                'cardiovascular|shock|vasopressors|epinephrine > 0.1 micrograms/kg/min',
                'cardiovascular|ventricular dysfunction|inotropic agent|epinephrine > 0.1 micrograms/kg/min',
                'cardiovascular|shock|inotropic agent|epinephrine > 0.1 micrograms/kg/min',
                'cardiovascular|myocardial ischemia / infarction|inotropic agent|epinephrine > 0.1 micrograms/kg/min',
                -- 肾上腺素（低剂量）
                'toxicology|drug overdose|vasopressors|epinephrine <= 0.1 micrograms/kg/min',
                'surgery|cardiac therapies|vasopressors|epinephrine <= 0.1 micrograms/kg/min',
                'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|epinephrine <= 0.1 micrograms/kg/min',
                'cardiovascular|shock|vasopressors|epinephrine <= 0.1 micrograms/kg/min',
                'cardiovascular|ventricular dysfunction|inotropic agent|epinephrine <= 0.1 micrograms/kg/min',
                'cardiovascular|shock|inotropic agent|epinephrine <= 0.1 micrograms/kg/min',
                'cardiovascular|myocardial ischemia / infarction|inotropic agent|epinephrine <= 0.1 micrograms/kg/min',
                -- 多巴胺（中剂量）
                'toxicology|drug overdose|vasopressors|dopamine  5-15 micrograms/kg/min',
                'surgery|cardiac therapies|vasopressors|dopamine  5-15 micrograms/kg/min',
                'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|dopamine  5-15 micrograms/kg/min',
                'cardiovascular|shock|vasopressors|dopamine  5-15 micrograms/kg/min',
                'surgery|cardiac therapies|inotropic agent|dopamine  5-15 micrograms/kg/min',
                'cardiovascular|ventricular dysfunction|inotropic agent|dopamine  5-15 micrograms/kg/min',
                'cardiovascular|myocardial ischemia / infarction|inotropic agent|dopamine  5-15 micrograms/kg/min',
                -- 多巴胺（高剂量）
                'toxicology|drug overdose|vasopressors|dopamine >15 micrograms/kg/min',
                'surgery|cardiac therapies|vasopressors|dopamine >15 micrograms/kg/min',
                'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|dopamine >15 micrograms/kg/min',
                'cardiovascular|shock|vasopressors|dopamine >15 micrograms/kg/min',
                'surgery|cardiac therapies|inotropic agent|dopamine >15 micrograms/kg/min',
                'cardiovascular|ventricular dysfunction|inotropic agent|dopamine >15 micrograms/kg/min',
                'cardiovascular|myocardial ischemia / infarction|inotropic agent|dopamine >15 micrograms/kg/min',
                -- 多巴胺（低剂量）
                'toxicology|drug overdose|agent specific therapy|beta blockers overdose|dopamine',
                'surgery|cardiac therapies|inotropic agent|dopamine <= 5 micrograms/kg/min',
                'cardiovascular|ventricular dysfunction|inotropic agent|dopamine <= 5 micrograms/kg/min',
                'cardiovascular|myocardial ischemia / infarction|inotropic agent|dopamine <= 5 micrograms/kg/min'
            ) THEN 1 
            ELSE 0 
        END)::SMALLINT AS vasopressor
    FROM treatment
    WHERE treatmentoffset >= 0 AND treatmentoffset < 1440  -- 第一天
    GROUP BY patientunitstayid, treatmentoffset
)
SELECT
    patientunitstayid,
    chartoffset,
    vasopressor
FROM tr
WHERE vasopressor = 1
ORDER BY patientunitstayid, chartoffset;
```

### 2. 提取使用升压药的患者（宽表格式）

```sql
-- 提取使用升压药的患者（每个患者一行）
SELECT
    patientunitstayid,
    MIN(treatmentoffset) AS first_vasopressor_offset,  -- 第一次使用升压药的时间
    MAX(treatmentoffset) AS last_vasopressor_offset,   -- 最后一次使用升压药的时间
    COUNT(*) AS num_vasopressor_records               -- 升压药记录数
FROM (
    SELECT
        patientunitstayid,
        treatmentoffset,
        MAX(CASE 
            WHEN treatmentstring IN (
                -- 省略完整的 treatmentstring 列表 ...
            ) THEN 1 
            ELSE 0 
        END)::SMALLINT AS vasopressor
    FROM treatment
    GROUP BY patientunitstayid, treatmentoffset
) tr
WHERE vasopressor = 1
GROUP BY patientunitstayid
ORDER BY patientunitstayid;
```

### 3. 筛选使用高剂量去甲肾上腺素的患者

```sql
-- 筛选使用高剂量去甲肾上腺素（>0.1 mcg/kg/min）的患者
SELECT DISTINCT patientunitstayid
FROM treatment
WHERE treatmentstring IN (
    'toxicology|drug overdose|vasopressors|norepinephrine > 0.1 micrograms/kg/min',
    'surgery|cardiac therapies|vasopressors|norepinephrine > 0.1 micrograms/kg/min',
    'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|norepinephrine > 0.1 micrograms/kg/min',
    'cardiovascular|shock|vasopressors|norepinephrine > 0.1 micrograms/kg/min',
    'cardiovascular|ventricular dysfunction|inotropic agent|norepinephrine > 0.1 micrograms/kg/min',
    'cardiovascular|shock|inotropic agent|norepinephrine > 0.1 micrograms/kg/min',
    'cardiovascular|myocardial ischemia / infarction|inotropic agent|norepinephrine > 0.1 micrograms/kg/min'
)
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

# 升压药的 treatmentstring 列表（基于 eicu-code）
VASOPRESSOR_TREATMENTS = [
    # 血管加压素
    'toxicology|drug overdose|vasopressors|vasopressin',
    'surgery|cardiac therapies|vasopressors|vasopressin',
    'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|vasopressin',
    'gastrointestinal|medications|hormonal therapy (for varices)|vasopressin',
    'cardiovascular|shock|vasopressors|vasopressin',
    # 苯肾上腺素
    'toxicology|drug overdose|vasopressors|phenylephrine (Neosynephrine)',
    'surgery|cardiac therapies|vasopressors|phenylephrine (Neosynephrine)',
    'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|phenylephrine (Neosynephrine)',
    'cardiovascular|shock|vasopressors|phenylephrine (Neosynephrine)',
    # 去甲肾上腺素（高剂量）
    'toxicology|drug overdose|vasopressors|norepinephrine > 0.1 micrograms/kg/min',
    'surgery|cardiac therapies|vasopressors|norepinephrine > 0.1 micrograms/kg/min',
    'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|norepinephrine > 0.1 micrograms/kg/min',
    'cardiovascular|shock|vasopressors|norepinephrine > 0.1 micrograms/kg/min',
    'cardiovascular|ventricular dysfunction|inotropic agent|norepinephrine > 0.1 micrograms/kg/min',
    'cardiovascular|shock|inotropic agent|norepinephrine > 0.1 micrograms/kg/min',
    'cardiovascular|myocardial ischemia / infarction|inotropic agent|norepinephrine > 0.1 micrograms/kg/min',
    # 去甲肾上腺素（低剂量）
    'toxicology|drug overdose|vasopressors|norepinephrine <= 0.1 micrograms/kg/min',
    'surgery|cardiac therapies|vasopressors|norepinephrine <= 0.1 micrograms/kg/min',
    'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|norepinephrine <= 0.1 micrograms/kg/min',
    'cardiovascular|shock|vasopressors|norepinephrine <= 0.1 micrograms/kg/min',
    'cardiovascular|ventricular dysfunction|inotropic agent|norepinephrine <= 0.1 micrograms/kg/min',
    'cardiovascular|shock|inotropic agent|norepinephrine <= 0.1 micrograms/kg/min',
    'cardiovascular|myocardial ischemia / infarction|inotropic agent|norepinephrine <= 0.1 micrograms/kg/min',
    # 肾上腺素（高剂量）
    'toxicology|drug overdose|vasopressors|epinephrine > 0.1 micrograms/kg/min',
    'surgery|cardiac therapies|vasopressors|epinephrine > 0.1 micrograms/kg/min',
    'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|epinephrine > 0.1 micrograms/kg/min',
    'cardiovascular|shock|vasopressors|epinephrine > 0.1 micrograms/kg/min',
    'cardiovascular|ventricular dysfunction|inotropic agent|epinephrine > 0.1 micrograms/kg/min',
    'cardiovascular|shock|inotropic agent|epinephrine > 0.1 micrograms/kg/min',
    'cardiovascular|myocardial ischemia / infarction|inotropic agent|epinephrine > 0.1 micrograms/kg/min',
    # 肾上腺素（低剂量）
    'toxicology|drug overdose|vasopressors|epinephrine <= 0.1 micrograms/kg/min',
    'surgery|cardiac therapies|vasopressors|epinephrine <= 0.1 micrograms/kg/min',
    'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|epinephrine <= 0.1 micrograms/kg/min',
    'cardiovascular|shock|vasopressors|epinephrine <= 0.1 micrograms/kg/min',
    'cardiovascular|ventricular dysfunction|inotropic agent|epinephrine <= 0.1 micrograms/kg/min',
    'cardiovascular|shock|inotropic agent|epinephrine <= 0.1 micrograms/kg/min',
    'cardiovascular|myocardial ischemia / infarction|inotropic agent|epinephrine <= 0.1 micrograms/kg/min',
    # 多巴胺（中剂量）
    'toxicology|drug overdose|vasopressors|dopamine  5-15 micrograms/kg/min',
    'surgery|cardiac therapies|vasopressors|dopamine  5-15 micrograms/kg/min',
    'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|dopamine  5-15 micrograms/kg/min',
    'cardiovascular|shock|vasopressors|dopamine  5-15 micrograms/kg/min',
    'surgery|cardiac therapies|inotropic agent|dopamine  5-15 micrograms/kg/min',
    'cardiovascular|ventricular dysfunction|inotropic agent|dopamine  5-15 micrograms/kg/min',
    'cardiovascular|myocardial ischemia / infarction|inotropic agent|dopamine  5-15 micrograms/kg/min',
    # 多巴胺（高剂量）
    'toxicology|drug overdose|vasopressors|dopamine >15 micrograms/kg/min',
    'surgery|cardiac therapies|vasopressors|dopamine >15 micrograms/kg/min',
    'neurologic|therapy for controlling cerebral perfusion pressure|vasopressors|dopamine >15 micrograms/kg/min',
    'cardiovascular|shock|vasopressors|dopamine >15 micrograms/kg/min',
    'surgery|cardiac therapies|inotropic agent|dopamine >15 micrograms/kg/min',
    'cardiovascular|ventricular dysfunction|inotropic agent|dopamine >15 micrograms/kg/min',
    'cardiovascular|myocardial ischemia / infarction|inotropic agent|dopamine >15 micrograms/kg/min',
    # 多巴胺（低剂量）
    'toxicology|drug overdose|agent specific therapy|beta blockers overdose|dopamine',
    'surgery|cardiac therapies|inotropic agent|dopamine <= 5 micrograms/kg/min',
    'cardiovascular|ventricular dysfunction|inotropic agent|dopamine <= 5 micrograms/kg/min',
    'cardiovascular|myocardial ischemia / infarction|inotropic agent|dopamine <= 5 micrograms/kg/min'
]

def extract_first_day_vasopressors(patient_ids=None):
    """
    从 treatment 表提取 ICU 第一天（0-1440 分钟）的升压药治疗
    （基于 eicu-code 官方实现）
    
    Parameters:
    -----------
    patient_ids : list or None
        患者 ID 列表，None 表示所有患者
    
    Returns:
    --------
    df : pandas DataFrame
        长格式升压药治疗数据
    """
    # 构建查询
    treatment_list_str = "','".join(VASOPRESSOR_TREATMENTS)
    query = f"""
        WITH tr AS (
            SELECT
                patientunitstayid,
                treatmentoffset AS chartoffset,
                MAX(CASE 
                    WHEN treatmentstring IN ('{treatment_list_str}')
                    THEN 1 
                    ELSE 0 
                END)::SMALLINT AS vasopressor
            FROM treatment
            WHERE treatmentoffset >= 0 AND treatmentoffset < 1440
            GROUP BY patientunitstayid, treatmentoffset
        )
        SELECT
            patientunitstayid,
            chartoffset,
            vasopressor
        FROM tr
        WHERE vasopressor = 1
        ORDER BY patientunitstayid, chartoffset
    """
    
    # 添加患者 ID 过滤
    if patient_ids is not None:
        if len(patient_ids) == 1:
            query = query.replace(
                "WHERE treatmentoffset",
                f"WHERE patientunitstayid = {patient_ids[0]} AND treatmentoffset"
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
                "FROM treatment",
                "FROM treatment, target_patients tp WHERE patientunitstayid = tp.pid"
            )
    
    # 执行查询
    df = pd.read_sql_query(query, conn)
    
    return df

def aggregate_vasopressors_to_wide(df):
    """
    将长格式升压药治疗数据转换为宽表格式
    
    Parameters:
    -----------
    df : pandas DataFrame
        长格式数据（来自 extract_first_day_vasopressors）
    
    Returns:
    --------
    wide_df : pandas DataFrame
        宽表格式，每个患者一行
    """
    # 计算统计量
    agg_df = df.groupby('patientunitstayid').agg(
        first_vasopressor_offset=('chartoffset', 'min'),
        last_vasopressor_offset=('chartoffset', 'max'),
        num_vasopressor_records=('vasopressor', 'count')
    ).reset_index()
    
    return agg_df

# 使用示例
if __name__ == "__main__":
    # 示例 1：提取所有患者的升压药治疗
    vaso = extract_first_day_vasopressors()
    print(f"提取了 {len(vaso)} 条记录")
    print(vaso.head(10))
    
    # 示例 2：提取特定患者的升压药治疗
    vaso = extract_first_day_vasopressors(
        patient_ids=[154347, 112001, 189456]
    )
    
    # 示例 3：转换为宽表
    wide_df = aggregate_vasopressors_to_wide(vaso)
    print("\n宽表格式:")
    print(wide_df.head(5))
    
    conn.close()
```

---

## 注意事项

### 1. 使用精确匹配（重要！）

✅ **正确** (精确匹配):
```sql
WHERE treatmentstring IN (
    'cardiovascular|shock|vasopressors|vasopressin',
    'cardiovascular|shock|vasopressors|phenylephrine (Neosynephrine)',
    -- ... 其他 treatmentstring ...
)
```

❌ **错误** (模糊匹配):
```sql
WHERE treatmentstring ILIKE '%vasopressor%'
```
（可能匹配到非升压药治疗记录）

### 2. 剂量分类（重要！）

升压药的剂量分类对于研究非常重要：

| 药物 | 低剂量 | 高剂量 |
|------|--------|--------|
| **去甲肾上腺素** | ≤0.1 mcg/kg/min | >0.1 mcg/kg/min |
| **肾上腺素** | ≤0.1 mcg/kg/min | >0.1 mcg/kg/min |
| **多巴胺** | ≤5 mcg/kg/min | 5-15 mcg/kg/min | >15 mcg/kg/min |

### 3. 性能优化

- `treatment` 表有 **大量记录**（具体数量未查，但应该很大）
- 查询时务必:
  - 使用 `patientunitstayid` 过滤
  - 限定 `treatmentoffset` 范围
  - 使用 `treatmentstring` 过滤（如 `treatmentstring IN (...)`）
  - 添加 `LIMIT` 子句（测试时）
  - 使用 `EXPLAIN` 分析查询计划

### 4. 时间偏移量

- `treatmentoffset`: 记录时间（相对于 ICU 入住的分钟数）
- 第一天 = `treatmentoffset >= 0 AND treatmentoffset < 1440`

---

## 常用查询场景

### 1. 筛选使用任何升压药的患者

```sql
-- 筛选使用任何升压药的患者
SELECT DISTINCT patientunitstayid
FROM treatment
WHERE treatmentstring IN (
    -- 省略完整的 treatmentstring 列表 ...
)
ORDER BY patientunitstayid;
```

### 2. 计算升压药使用随时间的变化

```sql
-- 计算升压药使用随时间的变化（ICU 第一天，每 6 小时）
WITH vaso_data AS (
    SELECT
        patientunitstayid,
        treatmentoffset,
        MAX(CASE 
            WHEN treatmentstring IN (
                -- 省略完整的 treatmentstring 列表 ...
            ) THEN 1 
            ELSE 0 
        END)::SMALLINT AS vasopressor
    FROM treatment
    WHERE treatmentoffset >= 0 AND treatmentoffset < 1440
    GROUP BY patientunitstayid, treatmentoffset
)
SELECT
    patientunitstayid,
    FLOOR(treatmentoffset / 360) * 360 AS time_window_start,  -- 6 小时窗口
    SUM(vasopressor) AS num_vasopressor_records
FROM vaso_data
WHERE vasopressor = 1
GROUP BY patientunitstayid, FLOOR(treatmentoffset / 360)
ORDER BY patientunitstayid, time_window_start;
```

---

## 参考链接

- **treatment 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/treatment.html
- **eicu-code 官方实现**: `D:\Project\eicu-code\concepts\pivoted\pivoted-treatment-vasopressor.sql`
- **eICU 数据介绍**: https://eicu-crd.mit.edu/

---

**最后更新**: 2026-05-22  
**更新人**: 悟空（基于 eicu-code 官方实现）
