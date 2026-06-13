# eICU 2.0 诊断与合并症查询

## 目录

1. [查询患者诊断](#查询患者诊断)
2. [既往史查询](#既往史查询)
3. [SQL 模板](#sql-模板)
4. [Python 模板](#python-模板)
5. [注意事项](#注意事项)

---

## 查询患者诊断

```sql
-- 查询患者的所有诊断
SELECT
    d.patientunitstayid,
    d.diagnosisstring,
    d.diagnosispriority,
    d.icd9code,
    d.diagnosisoffset,
    d.activeupondischarge
FROM diagnosis d
WHERE d.patientunitstayid = %(patient_id)s
ORDER BY d.diagnosispriority, d.diagnosisoffset;
```

**字段说明**:
- `diagnosisstring`: 诊断名称（主要字段）
- `diagnosispriority`: 诊断优先级（`'Primary'` = 主诊断，`'Secondary'` = 次要诊断）
- `icd9code`: ICD-9 编码（可能为 NULL）
- `activeupondischarge`: 出院时是否仍活动（`'true'` / `'false'`）
- `diagnosisoffset`: 诊断记录相对于 ICU 入住的时间偏移（分钟）

> **注意**: `icd9code` 在大部分患者中为 NULL。eICU 的诊断主要依赖 `diagnosisstring` 文本匹配。

---

## 既往史查询

`pasthistory` 表可用于识别合并症。

```sql
-- 查看既往史的所有不重复描述
SELECT DISTINCT pasthistorystring, COUNT(*) AS cnt
FROM pasthistory
GROUP BY pasthistorystring
ORDER BY cnt DESC
LIMIT 100;

-- 常见的 pasthistorystring 示例:
-- "renal dialysis", "diabetes", "COPD", "congestive heart failure",
-- "liver disease", "immunocompromised", "vascular disease", etc.

-- 查询特定患者的既往史
SELECT 
    ph.patientunitstayid,
    ph.pasthistoryoffset,
    ph.pasthistorystring,
    ph.icd9code
FROM pasthistory ph
WHERE ph.patientunitstayid = %(patient_id)s
ORDER BY ph.pasthistoryoffset;
```

**字段说明**:
- `pasthistorystring`: 既往史描述（主要字段）
- `pasthistoryoffset`: 记录时间偏移量（分钟）
- `icd9code`: ICD-9 编码（可能为 NULL）

---

## SQL 模板

### 1. 按诊断名称筛选患者

```sql
-- 筛选特定诊断的患者（如脓毒症）
SELECT DISTINCT d.patientunitstayid
FROM diagnosis d
WHERE d.diagnosisstring ILIKE '%sepsis%'
   OR d.diagnosisstring ILIKE '%septic%'
   OR d.diagnosisstring ILIKE '%infection%'
ORDER BY d.patientunitstayid;
```

### 2. 合并症筛选（基于既往史）

```sql
-- 糖尿病
SELECT DISTINCT ph.patientunitstayid
FROM pasthistory ph
WHERE ph.pasthistorystring ILIKE '%diabetes%'
   OR ph.pasthistorystring ILIKE '%dm%'

UNION ALL

-- 慢性肺病（COPD）
SELECT DISTINCT ph.patientunitstayid
FROM pasthistory ph
WHERE ph.pasthistorystring ILIKE '%copd%'
   OR ph.pasthistorystring ILIKE '%chronic obstructive%'
   OR ph.pasthistorystring ILIKE '%pulmonary disease%'

UNION ALL

-- 肾病（透析）
SELECT DISTINCT ph.patientunitstayid
FROM pasthistory ph
WHERE ph.pasthistorystring ILIKE '%renal%'
   OR ph.pasthistorystring ILIKE '%dialysis%'
   OR ph.pasthistorystring ILIKE '%esrd%'

UNION ALL

-- 心脏疾病
SELECT DISTINCT ph.patientunitstayid
FROM pasthistory ph
WHERE ph.pasthistorystring ILIKE '%heart failure%'
   OR ph.pasthistorystring ILIKE '%chf%'
   OR ph.pasthistorystring ILIKE '%cardiomyopathy%'
   OR ph.pasthistorystring ILIKE '%cad%'
   OR ph.pasthistorystring ILIKE '%coronary%'

UNION ALL

-- 肝病
SELECT DISTINCT ph.patientunitstayid
FROM pasthistory ph
WHERE ph.pasthistorystring ILIKE '%liver%'
   OR ph.pasthistorystring ILIKE '%hepatic%'
   OR ph.pasthistorystring ILIKE '%cirrhosis%'

UNION ALL

-- 免疫低下
SELECT DISTINCT ph.patientunitstayid
FROM pasthistory ph
WHERE ph.pasthistorystring ILIKE '%immunocompromised%'
   OR ph.pasthistorystring ILIKE '%immunosuppress%'
   OR ph.pasthistorystring ILIKE '%hiv%'
   OR ph.pasthistorystring ILIKE '%aids%';
```

### 3. 筛选脓毒症患者（基于诊断 + 微生物培养）

```sql
-- 方法 1: 基于诊断名称
SELECT DISTINCT p.patientunitstayid
FROM patient p
INNER JOIN diagnosis d 
    ON p.patientunitstayid = d.patientunitstayid
WHERE d.diagnosisstring ILIKE '%sepsis%'
   OR d.diagnosisstring ILIKE '%septic%'
   OR d.diagnosisstring ILIKE '%infection%'
ORDER BY p.patientunitstayid;

-- 方法 2: 基于微生物培养（更严格）
SELECT DISTINCT p.patientunitstayid
FROM patient p
INNER JOIN microlab m 
    ON p.patientunitstayid = m.patientunitstayid
WHERE m.organism IS NOT NULL 
  AND m.organism != ''
ORDER BY p.patientunitstayid;

-- 方法 3: 结合两者（推荐）
SELECT DISTINCT p.patientunitstayid
FROM patient p
WHERE EXISTS (
    SELECT 1 FROM diagnosis d
    WHERE d.patientunitstayid = p.patientunitstayid
      AND (d.diagnosisstring ILIKE '%sepsis%'
           OR d.diagnosisstring ILIKE '%septic%')
)
OR EXISTS (
    SELECT 1 FROM microlab m
    WHERE m.patientunitstayid = p.patientunitstayid
      AND m.organism IS NOT NULL 
      AND m.organism != ''
)
ORDER BY p.patientunitstayid;
```

### 4. 提取主诊断（diagnosispriority = 'Primary'）

```sql
SELECT
    d.patientunitstayid,
    d.diagnosisstring AS primary_diagnosis,
    d.icd9code,
    d.activeupondischarge
FROM diagnosis d
WHERE d.diagnosispriority = 'Primary'
ORDER BY d.patientunitstayid;
```

### 5. 统计每个患者的诊断数量

```sql
SELECT
    d.patientunitstayid,
    COUNT(*) AS num_diagnoses,
    MAX(CASE WHEN d.diagnosispriority = 'Primary' 
              THEN d.diagnosisstring 
              ELSE NULL END) AS primary_diagnosis,
    STRING_AGG(DISTINCT d.diagnosisstring, '; ') AS all_diagnoses
FROM diagnosis d
GROUP BY d.patientunitstayid
ORDER BY num_diagnoses DESC
LIMIT 100;
```

---

## Python 模板

```python
import psycopg2
import pandas as pd

# 数据库连接
conn = psycopg2.connect(
    host="your_host",
    port=5432,
    dbname="eicu",
    user="your_user",
    password="your_password"
)

def get_diagnoses(patient_id):
    """
    获取患者的所有诊断
    
    Parameters:
    -----------
    patient_id : int
        患者 ID (patientunitstayid)
    
    Returns:
    --------
    df : pandas DataFrame
        诊断记录
    """
    query = """
        SELECT 
            diagnosisstring,
            diagnosispriority,
            icd9code,
            diagnosisoffset,
            activeupondischarge
        FROM diagnosis
        WHERE patientunitstayid = %(pid)s
        ORDER BY diagnosispriority, diagnosisoffset
    """
    df = pd.read_sql_query(query, conn, params={"pid": patient_id})
    return df

def get_past_history(patient_id):
    """
    获取患者的既往史
    
    Parameters:
    -----------
    patient_id : int
        患者 ID (patientunitstayid)
    
    Returns:
    --------
    df : pandas DataFrame
        既往史记录
    """
    query = """
        SELECT 
            pasthistoryoffset,
            pasthistorystring,
            icd9code
        FROM pasthistory
        WHERE patientunitstayid = %(pid)s
        ORDER BY pasthistoryoffset
    """
    df = pd.read_sql_query(query, conn, params={"pid": patient_id})
    return df

def find_patients_by_diagnosis(keywords, match_type='any'):
    """
    根据诊断关键词查找患者
    
    ⚠️ 安全警告：此函数使用 f-string 插值构建 SQL，存在 SQL 注入风险。
    仅用于受信任的输入。生产环境应使用参数化查询。
    
    Parameters:
    -----------
    keywords : list
        诊断关键词列表（如 ['sepsis', 'septic']）
        注意：关键词会被直接插入 SQL，请确保来源可信
    match_type : str
        匹配类型：'any' (任一关键词) 或 'all' (所有关键词)
    
    Returns:
    --------
    df : pandas DataFrame
        匹配的患者 ID 列表
    """
    # ⚠️ 安全风险：此实现使用字符串插值，仅用于示例
    # 生产环境应使用参数化查询或预定义关键词白名单
    if match_type == 'any':
        where_clauses = " OR ".join([
            f"d.diagnosisstring ILIKE '%{kw}%'" for kw in keywords
        ])
    elif match_type == 'all':
        where_clauses = " AND ".join([
            f"d.diagnosisstring ILIKE '%{kw}%'" for kw in keywords
        ])
    
    query = f"""
        SELECT DISTINCT d.patientunitstayid
        FROM diagnosis d
        WHERE {where_clauses}
        ORDER BY d.patientunitstayid
    """
    df = pd.read_sql_query(query, conn)
    return df

    # 更安全的实现示例（使用参数化查询）:
    # def find_patients_by_diagnosis_safe(keywords, match_type='any'):
    #     if match_type == 'any':
    #         where_clauses = " OR ".join(["d.diagnosisstring ILIKE %s" for _ in keywords])
    #     else:
    #         where_clauses = " AND ".join(["d.diagnosisstring ILIKE %s" for _ in keywords])
    #     
    #     query = f"""
    #         SELECT DISTINCT d.patientunitstayid
    #         FROM diagnosis d
    #         WHERE {where_clauses}
    #         ORDER BY d.patientunitstayid
    #     """
    #     params = [f"%{kw}%" for kw in keywords]
    #     df = pd.read_sql_query(query, conn, params=params)
    #     return df

def find_patients_by_past_history(keywords):
    """
    根据既往史关键词查找患者
    
    ⚠️ 安全警告：此函数使用 f-string 插值构建 SQL，存在 SQL 注入风险。
    仅用于受信任的输入。生产环境应使用参数化查询。
    
    Parameters:
    -----------
    keywords : list
        既往史关键词列表（如 ['diabetes', 'copd']）
        注意：关键词会被直接插入 SQL，请确保来源可信
    
    Returns:
    --------
    df : pandas DataFrame
        匹配的患者 ID 列表
    """
    # ⚠️ 安全风险：此实现使用字符串插值，仅用于示例
    # 生产环境应使用参数化查询或预定义关键词白名单
    where_clauses = " OR ".join([
        f"ph.pasthistorystring ILIKE '%{kw}%'" for kw in keywords
    ])
    
    query = f"""
        SELECT DISTINCT ph.patientunitstayid
        FROM pasthistory ph
        WHERE {where_clauses}
        ORDER BY ph.patientunitstayid
    """
    df = pd.read_sql_query(query, conn)
    return df

    # 更安全的实现示例（使用参数化查询）:
    # def find_patients_by_past_history_safe(keywords):
    #     where_clauses = " OR ".join(["ph.pasthistorystring ILIKE %s" for _ in keywords])
    #     
    #     query = f"""
    #         SELECT DISTINCT ph.patientunitstayid
    #         FROM pasthistory ph
    #         WHERE {where_clauses}
    #         ORDER BY ph.patientunitstayid
    #     """
    #     params = [f"%{kw}%" for kw in keywords]
    #     df = pd.read_sql_query(query, conn, params=params)
    #     return df

def get_comorbidities(patient_ids=None):
    """
    提取患者的合并症（基于既往史）
    
    Parameters:
    -----------
    patient_ids : list or None
        患者 ID 列表，None 表示所有患者
    
    Returns:
    --------
    df : pandas DataFrame
        每个患者的合并症标记（宽表格式）
    """
    query = """
        SELECT 
            ph.patientunitstayid,
            MAX(CASE WHEN ph.pasthistorystring ILIKE '%diabetes%' 
                     THEN 1 ELSE 0 END) AS comorbidity_diabetes,
            MAX(CASE WHEN ph.pasthistorystring ILIKE '%copd%' 
                     OR ph.pasthistorystring ILIKE '%chronic obstructive%' 
                     THEN 1 ELSE 0 END) AS comorbidity_copd,
            MAX(CASE WHEN ph.pasthistorystring ILIKE '%renal%' 
                     OR ph.pasthistorystring ILIKE '%dialysis%' 
                     THEN 1 ELSE 0 END) AS comorbidity_renal,
            MAX(CASE WHEN ph.pasthistorystring ILIKE '%heart failure%' 
                     OR ph.pasthistorystring ILIKE '%chf%' 
                     THEN 1 ELSE 0 END) AS comorbidity_chf,
            MAX(CASE WHEN ph.pasthistorystring ILIKE '%liver%' 
                     OR ph.pasthistorystring ILIKE '%hepatic%' 
                     THEN 1 ELSE 0 END) AS comorbidity_liver,
            MAX(CASE WHEN ph.pasthistorystring ILIKE '%immunocompromised%' 
                     THEN 1 ELSE 0 END) AS comorbidity_immunocompromised
        FROM pasthistory ph
        GROUP BY ph.patientunitstayid
        ORDER BY ph.patientunitstayid
    """
    
    if patient_ids is not None:
        if len(patient_ids) == 1:
            query = query.replace(
                "ORDER BY", 
                f"WHERE ph.patientunitstayid = {patient_ids[0]} GROUP BY"
            )
        else:
            ids_str = ",".join(map(str, patient_ids))
            query = f"""
                WITH target_patients AS (
                    SELECT unnest(ARRAY[{ids_str}]) AS pid
                )
                SELECT 
                    ph.patientunitstayid,
                    MAX(CASE WHEN ph.pasthistorystring ILIKE '%diabetes%' 
                             THEN 1 ELSE 0 END) AS comorbidity_diabetes,
                    MAX(CASE WHEN ph.pasthistorystring ILIKE '%copd%' 
                             THEN 1 ELSE 0 END) AS comorbidity_copd,
                    MAX(CASE WHEN ph.pasthistorystring ILIKE '%renal%' 
                             THEN 1 ELSE 0 END) AS comorbidity_renal,
                    MAX(CASE WHEN ph.pasthistorystring ILIKE '%heart failure%' 
                             THEN 1 ELSE 0 END) AS comorbidity_chf,
                    MAX(CASE WHEN ph.pasthistorystring ILIKE '%liver%' 
                             THEN 1 ELSE 0 END) AS comorbidity_liver,
                    MAX(CASE WHEN ph.pasthistorystring ILIKE '%immunocompromised%' 
                             THEN 1 ELSE 0 END) AS comorbidity_immunocompromised
                FROM pasthistory ph, target_patients tp
                WHERE ph.patientunitstayid = tp.pid
                GROUP BY ph.patientunitstayid
                ORDER BY ph.patientunitstayid
            """
    
    df = pd.read_sql_query(query, conn)
    return df

# 使用示例
if __name__ == "__main__":
    # 示例 1：获取特定患者的诊断
    patient_id = 142289
    dx = get_diagnoses(patient_id)
    print(f"患者 {patient_id} 的诊断:")
    print(dx)
    
    # 示例 2：获取特定患者的既往史
    ph = get_past_history(patient_id)
    print(f"\n患者 {patient_id} 的既往史:")
    print(ph)
    
    # 示例 3：查找脓毒症患者
    sepsis_patients = find_patients_by_diagnosis(['sepsis', 'septic'])
    print(f"\n找到 {len(sepsis_patients)} 例脓毒症患者")
    print(sepsis_patients.head(10))
    
    # 示例 4：提取合并症
    comorbidities = get_comorbidities(sepsis_patients['patientunitstayid'].tolist()[:100])
    print(f"\n合并症统计 (前 100 例患者):")
    print(comorbidities.describe())
    
    conn.close()
```

---

## 注意事项

### 1. 字段名规范

✅ **正确** (小写):
```sql
SELECT d.patientunitstayid, d.diagnosisstring
FROM diagnosis d
WHERE d.diagnosispriority = 'Primary'
```

❌ **错误** (camelCase):
```sql
SELECT d.patientUnitStayID, d.diagnosisName  -- 字段不存在！
FROM diagnosis d
WHERE d.diagnosisPriority = 'Primary'
```

### 2. 诊断编码不统一

- **部分有 ICD-9**: `icd9code` 字段（但很多为 NULL）
- **部分只有文本**: `diagnosisstring` 字段（主要依赖）
- **建议**: 使用 `ILIKE` 进行模糊匹配

```sql
-- 推荐：同时使用 ICD-9 编码和文本匹配
SELECT DISTINCT d.patientunitstayid
FROM diagnosis d
WHERE d.icd9code ILIKE '%99592%'  -- ICD-9 for sepsis
   OR d.diagnosisstring ILIKE '%sepsis%'
```

### 3. 既往史结构更规范

- **pastHistory 表**: 比 MIMIC-IV 的 `diagnoses_icd` 更易于识别合并症
- **pasthistorystring**: 直接包含疾病名称（如 "diabetes", "COPD"）
- **建议**: 结合 `diagnosis` 和 `pasthistory` 表识别合并症

### 4. 模糊匹配必不可少

- `diagnosisstring` 和 `pasthistorystring` 的具体措辞因站点而异
- **建议工作流程**:
  1. 先查询 `SELECT DISTINCT diagnosisstring FROM diagnosis WHERE diagnosisstring ILIKE '%keyword%'`
  2. 根据实际值调整匹配模式
  3. 使用 `ILIKE` (不区分大小写) 而非 `LIKE`

### 5. 诊断优先级

- `diagnosispriority = 'Primary'`: 主诊断
- `diagnosispriority = 'Secondary'`: 次要诊断
- **建议**: 分析时优先考虑主诊断

### 6. 出院时活动状态

- `activeupondischarge = 'true'`: 出院时该诊断仍然活动
- `activeupondischarge = 'false'`: 出院时该诊断已不活动
- **用途**: 识别持续的疾病状态

---

## 常用查询场景

### 1. 筛选特定疾病的患者（如急性心肌梗死）

```sql
SELECT DISTINCT d.patientunitstayid
FROM diagnosis d
WHERE d.diagnosisstring ILIKE '%myocardial infarction%'
   OR d.diagnosisstring ILIKE '%ami%'
   OR d.diagnosisstring ILIKE '%stemi%'
   OR d.diagnosisstring ILIKE '%nstemi%'
ORDER BY d.patientunitstayid;
```

### 2. 统计合并症患病率

```sql
SELECT 
    'diabetes' AS comorbidity,
    COUNT(DISTINCT ph.patientunitstayid) AS num_patients,
    (COUNT(DISTINCT ph.patientunitstayid) * 100.0 / (SELECT COUNT(*) FROM patient)) AS prevalence_pct
FROM pasthistory ph
WHERE ph.pasthistorystring ILIKE '%diabetes%'

UNION ALL

SELECT 
    'copd' AS comorbidity,
    COUNT(DISTINCT ph.patientunitstayid) AS num_patients,
    (COUNT(DISTINCT ph.patientunitstayid) * 100.0 / (SELECT COUNT(*) FROM patient)) AS prevalence_pct
FROM pasthistory ph
WHERE ph.pasthistorystring ILIKE '%copd%'
   OR ph.pasthistorystring ILIKE '%chronic obstructive%'

ORDER BY num_patients DESC;
```

### 3. 提取 SOFA 评分所需的疾病信息

```sql
SELECT 
    p.patientunitstayid,
    -- 肝病（胆红素升高）
    MAX(CASE WHEN ph.pasthistorystring ILIKE '%liver%' 
             THEN 1 ELSE 0 END) AS has_liver_disease,
    -- 凝血病（血小板降低）
    MAX(CASE WHEN ph.pasthistorystring ILIKE '%coagulopathy%' 
             THEN 1 ELSE 0 END) AS has_coagulopathy,
    -- 心血管病（低血压）
    MAX(CASE WHEN d.diagnosisstring ILIKE '%shock%' 
             THEN 1 ELSE 0 END) AS has_shock
FROM patient p
LEFT JOIN pasthistory ph
    ON p.patientunitstayid = ph.patientunitstayid
LEFT JOIN diagnosis d
    ON p.patientunitstayid = d.patientunitstayid
GROUP BY p.patientunitstayid
ORDER BY p.patientunitstayid;
```

---

## 参考链接

- **diagnosis 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/diagnosis.html
- **pasthistory 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/pasthistory.html

---

**最后更新**: 2026-05-20  
**更新人**: 悟空（基于 MIT 官方 SchemaSpy 文档修正）
