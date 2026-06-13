# eICU 2.0 药物查询

**参考**: 
- eICU 2.0 SchemaSpy 文档: https://lcp.mit.edu/eicu-schema-spy/
- `medication` 表

---

## 目录

1. [常用字段映射](#常用字段映射)
2. [SQL 模板](#sql-模板)
3. [Python 模板](#python-模板)
4. [注意事项](#注意事项)

---

## 常用字段映射

### 1. medication 表核心字段

| 字段名 | 类型 | 说明 | 示例值 |
|--------|------|------|--------|
| `patientunitstayid` | INTEGER | ICU 住院唯一标识符（外键） | 0, 1, 2, ... |
| `medicationoffset` | INTEGER | 给药时间偏移（分钟，相对于 ICU 入住） | 120, 480, 960, ... |
| `medicationname` | VARCHAR | 药物名称 | 'Heparin', 'Insulin', 'Vancomycin', ... |
| `dosage` | VARCHAR | 剂量 | '5000', '10', '1000', ... |
| `routeadmin` | VARCHAR | 给药途径 | 'IV', 'PO', 'SC', 'IM', ... |
| `frequency` | VARCHAR | 给药频率 | 'Daily', 'BID', 'TID', 'QID', ... |
| `infusionrate` | VARCHAR | 输注速率 | '2', '5', '10', ... |
| `infusionunit` | VARCHAR | 输注单位 | 'mL/hr', 'units/hr', ... |
| `category` | VARCHAR | 药物类别 | 'Antibiotic', 'Anticoagulant', ... |

⚠️ **注意**: `medication` 表字段名可能因 eICU 版本而异，请务必使用 `\\d medication` 或 SchemaSpy 确认。

### 2. 常用药物类别

| 类别 | 常见药物 | 说明 |
|------|----------|------|
| 抗生素 | Vancomycin, Ceftriaxone, Meropenem | 抗感染 |
| 抗凝药 | Heparin, Enoxaparin | 抗凝 |
| 降糖药 | Insulin | 控制血糖 |
| 升压药 | Norepinephrine, Epinephrine | 升血压 |
| 镇静药 | Propofol, Midazolam, Fentanyl | 镇静镇痛 |
| 利尿剂 | Furosemide | 利尿 |
| 激素 | Hydrocortisone, Prednisone | 抗炎/免疫抑制 |

---

## SQL 模板

### 1. 查询患者的所有药物

```sql
-- 查询患者的所有药物（前 100 条）
SELECT
    m.patientunitstayid,
    m.medicationoffset,
    m.medicationname,
    m.dosage,
    m.routeadmin,
    m.frequency,
    m.infusionrate,
    m.infusionunit
FROM medication m
WHERE m.patientunitstayid = %(patient_id)s
ORDER BY m.medicationoffset;
```

### 2. 筛选特定药物（如胰岛素）

```sql
-- 筛选使用胰岛素的患者
SELECT DISTINCT m.patientunitstayid
FROM medication m
WHERE m.medicationname ILIKE '%insulin%'
ORDER BY m.patientunitstayid;
```

⚠️ **注意**: `medicationname` 可能因站点而异，建议先查询确认：

```sql
-- 查看所有不重复的药物名称（前 100 个）
SELECT DISTINCT medicationname, COUNT(*) AS cnt
FROM medication
GROUP BY medicationname
ORDER BY cnt DESC
LIMIT 100;
```

### 3. 提取第一天使用的药物（0-1440 分钟）

```sql
-- 提取第一天使用的所有药物
SELECT
    m.patientunitstayid,
    m.medicationoffset,
    m.medicationname,
    m.dosage,
    m.routeadmin
FROM medication m
WHERE m.medicationoffset >= 0 AND m.medicationoffset < 1440
ORDER BY m.patientunitstayid, m.medicationoffset;
```

### 4. 统计每种药物的使用次数

```sql
-- 统计每种药物的使用次数（前 50 个）
SELECT
    m.medicationname,
    COUNT(*) AS num_uses,
    COUNT(DISTINCT m.patientunitstayid) AS num_patients
FROM medication m
GROUP BY m.medicationname
ORDER BY num_uses DESC
LIMIT 50;
```

### 5. 提取升压药使用（如去甲肾上腺素）

```sql
-- 提取使用升压药的患者
SELECT DISTINCT m.patientunitstayid
FROM medication m
WHERE m.medicationname ILIKE '%norepinephrine%'
   OR m.medicationname ILIKE '%epinephrine%'
   OR m.medicationname ILIKE '%phenylephrine%'
   OR m.medicationname ILIKE '%dopamine%'
ORDER BY m.patientunitstayid;
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

# 常用药物关键词
MEDICATION_KEYWORDS = {
    'insulin': ['insulin'],
    'heparin': ['heparin'],
    'vancomycin': ['vancomycin'],
    'norepinephrine': ['norepinephrine', 'levophed'],
    'epinephrine': ['epinephrine', 'adrenaline'],
    'propofol': ['propofol'],
    'furosemide': ['furosemide', 'lasix']
}

def extract_medications(patient_ids=None, med_keywords=None):
    """
    提取患者的药物使用记录
    
    Parameters:
    -----------
    patient_ids : list or None
        患者 ID 列表，None 表示所有患者
    med_keywords : dict or None
        要提取的药物及其关键词，None 表示全部
    
    Returns:
    --------
    df : pandas DataFrame
        药物使用记录
    """
    if med_keywords is None:
        med_keywords = MEDICATION_KEYWORDS
    
    # 构建 medicationname 过滤条件
    med_filters = []
    for med, keywords in med_keywords.items():
        for kw in keywords:
            med_filters.append(f"m.medicationname ILIKE '%{kw}%'")
    
    where_clause = " OR ".join(med_filters)
    
    # 构建查询
    query = f"""
        SELECT 
            m.patientunitstayid,
            m.medicationoffset,
            m.medicationname,
            m.dosage,
            m.routeadmin,
            m.frequency,
            m.infusionrate,
            m.infusionunit
        FROM medication m
        WHERE m.medicationoffset >= 0 AND m.medicationoffset < 1440
          AND ({where_clause})
        ORDER BY m.patientunitstayid, m.medicationoffset
    """
    
    # 添加患者 ID 过滤
    if patient_ids is not None:
        if len(patient_ids) == 1:
            query = query.replace(
                "WHERE m.medicationoffset",
                f"WHERE m.patientunitstayid = {patient_ids[0]} AND m.medicationoffset"
            )
        else:
            # 使用临时表
            ids_str = ",".join(map(str, patient_ids))
            query = f"""
                WITH target_patients AS (
                    SELECT unnest(ARRAY[{ids_str}]) AS pid
                )
                SELECT 
                    m.patientunitstayid,
                    m.medicationoffset,
                    m.medicationname,
                    m.dosage,
                    m.routeadmin,
                    m.frequency,
                    m.infusionrate,
                    m.infusionunit
                FROM medication m, target_patients tp
                WHERE m.patientunitstayid = tp.pid
                  AND m.medicationoffset >= 0 AND m.medicationoffset < 1440
                  AND ({where_clause})
                ORDER BY m.patientunitstayid, m.medicationoffset
            """
    
    # 执行查询
    df = pd.read_sql_query(query, conn)
    
    return df

def get_medication_stats(df):
    """
    统计药物使用情况
    
    Parameters:
    -----------
    df : pandas DataFrame
        药物使用记录（来自 extract_medications）
    
    Returns:
    --------
    stats_df : pandas DataFrame
        每种药物的统计信息
    """
    stats_df = df.groupby('medicationname').agg(
        num_uses=('patientunitstayid', 'count'),
        num_patients=('patientunitstayid', 'nunique')
    ).reset_index()
    
    stats_df = stats_df.sort_values('num_uses', ascending=False)
    
    return stats_df

# 使用示例
if __name__ == "__main__":
    # 示例 1：提取所有患者的胰岛素使用记录
    insulin = extract_medications(
        med_keywords={'insulin': ['insulin']}
    )
    print(f"提取了 {len(insulin)} 条胰岛素使用记录")
    print(insulin.head(10))
    
    # 示例 2：查看实际的药物名称
    cursor = conn.cursor()
    cursor.execute("""
        SELECT DISTINCT medicationname, COUNT(*) AS cnt
        FROM medication
        WHERE medicationname ILIKE '%insulin%'
        GROUP BY medicationname
        ORDER BY cnt DESC
        LIMIT 20
    """)
    print("\n实际的药物名称:")
    for row in cursor.fetchall():
        print(f"  {row[0]}: {row[1]} 次使用")
    cursor.close()
    
    # 示例 3：统计药物使用次数
    all_meds = extract_medications()
    stats = get_medication_stats(all_meds)
    print("\n药物使用统计（前 10）:")
    print(stats.head(10))
    
    conn.close()
```

---

## 注意事项

### 1. 字段名规范

✅ **正确** (小写):
```sql
SELECT m.patientunitstayid, m.medicationname, m.dosage
FROM medication m
WHERE m.medicationoffset >= 0
```

❌ **错误** (camelCase):
```sql
SELECT m.patientUnitStayID, m.medicationName, m.dosage  -- 字段不存在！
FROM medication m
WHERE m.medicationOffset >= 0
```

### 2. 时间偏移量

- `medicationoffset`: 给药时间（相对于 ICU 入住的分钟数）
- 第一天 = `medicationoffset >= 0 AND medicationoffset < 1440`

### 3. 药物名称的不确定性

- **不同站点可能使用不同的 medicationname**
- **建议工作流程**:
  1. 先查询 `SELECT DISTINCT medicationname FROM medication WHERE medicationname ILIKE '%keyword%'`
  2. 根据实际值调整过滤条件
  3. 使用 `ILIKE` 进行模糊匹配（不区分大小写）

### 4. 数据类型转换

- `dosage` 字段类型为 `VARCHAR`，计算时需转换：`CAST(m.dosage AS numeric)` 或 `m.dosage::numeric`
- 转换失败会得到 `NULL`（使用 `NULLIF` 或 `CASE WHEN` 处理异常值）

```sql
-- 安全转换示例
SELECT 
    medicationname,
    dosage,
    CASE 
        WHEN dosage ~ '^[0-9]+\.?[0-9]*$' 
        THEN dosage::numeric 
        ELSE NULL 
    END AS dosage_numeric
FROM medication
WHERE dosage IS NOT NULL
```

### 5. 数据质量

- **缺失值多**: eICU 的用药数据完整性可能较低
- **异常值**: 查询时考虑添加合理性检查
- **重复值**: 同一患者同一药物可能有多个记录（不同时间）

### 6. 性能优化

- `medication` 表可能有 **数百万条记录**
- 查询时务必:
  - 使用 `patientunitstayid` 过滤
  - 限定 `medicationoffset` 范围
  - 添加 `LIMIT` 子句（测试时）
  - 使用 `EXPLAIN` 分析查询计划

---

## 常用查询场景

### 1. 提取使用胰岛素的患者

```sql
SELECT DISTINCT m.patientunitstayid
FROM medication m
WHERE m.medicationname ILIKE '%insulin%'
ORDER BY m.patientunitstayid;
```

### 2. 提取使用升压药的患者

```sql
SELECT DISTINCT m.patientunitstayid
FROM medication m
WHERE m.medicationname ILIKE '%norepinephrine%'
   OR m.medicationname ILIKE '%epinephrine%'
   OR m.medicationname ILIKE '%phenylephrine%'
   OR m.medicationname ILIKE '%dopamine%'
ORDER BY m.patientunitstayid;
```

### 3. 计算 SOFA 评分所需的升压药剂量

```sql
-- 提取升压药使用记录（用于 SOFA 评分）
SELECT 
    m.patientunitstayid,
    m.medicationoffset,
    m.medicationname,
    m.dosage,
    m.infusionrate,
    m.infusionunit
FROM medication m
WHERE m.medicationoffset BETWEEN 0 AND 1440
  AND (
    m.medicationname ILIKE '%norepinephrine%'
    OR m.medicationname ILIKE '%epinephrine%'
    OR m.medicationname ILIKE '%dopamine%'
    OR m.medicationname ILIKE '%phenylephrine%'
  )
ORDER BY m.patientunitstayid, m.medicationoffset;
```

---

## 参考链接

- **medication 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/medication.html
- **eICU 数据介绍**: https://eicu-crd.mit.edu/

---

**最后更新**: 2026-05-22  
**更新人**: 悟空（基于 eICU 2.0 官方文档）
