# eICU 2.0 人口统计学查询

**参考文件**: 
- `D:\Project\eicu-code\concepts\basic_demographics.sql`
- `D:\Project\eicu-code\concepts\icustay_detail.sql`

---

## 目录

1. [常用字段映射](#常用字段映射)
2. [SQL 模板](#sql-模板)
3. [Python 模板](#python-模板)
4. [注意事项](#注意事项)

---

## 常用字段映射

### 1. patient 表核心字段

| 字段名 | 类型 | 说明 | 示例值 |
|--------|------|------|--------|
| `patientunitstayid` | INTEGER | ICU 住院唯一标识符（主键） | 0, 1, 2, ... |
| `uniquepid` | VARCHAR | 患者唯一标识符（跨住院） | '001-34521' |
| `age` | VARCHAR | 年龄（注意：可能是 '\> 89'） | '46', '> 89' |
| `gender` | VARCHAR | 性别 | 'Male', 'Female' |
| `ethnicity` | VARCHAR | 种族 | 'Caucasian', 'African American', ... |
| `hospitaldischargestatus` | VARCHAR | 出院状态 | 'Alive', 'Expired' |
| `hospitaldischargeoffset` | INTEGER | 出院时间偏移（分钟） | 4320, 5760, ... |
| `unitdischargeoffset` | INTEGER | ICU 出院时间偏移（分钟） | 1440, 2880, ... |
| `hospitalid` | INTEGER | 医院 ID | 164, 188, ... |
| `unittype` | VARCHAR | ICU 类型 | 'MICU', 'SICU', 'CCU', ... |
| `admissionheight` | NUMERIC | 入院身高（cm） | 170.2, 165.8, ... |
| `admissionweight` | NUMERIC | 入院体重（kg） | 75.3, 82.1, ... |
| `dischargeweight` | NUMERIC | 出院体重（kg） | 73.5, 80.2, ... |
| `apacheadmissiondx` | VARCHAR | APACHE IV 入院诊断 | 'Sepsis', 'Pneumonia', ... |

### 2. 衍生字段（基于 eicu-code）

| 衍生字段 | 计算方式 | 说明 |
|----------|----------|------|
| `gender_numeric` | `CASE WHEN gender = 'Male' THEN 1 WHEN gender = 'Female' THEN 2 ELSE NULL END` | 性别数值化（1=男，2=女） |
| `hosp_mortality_numeric` | `CASE WHEN hospitaldischargestatus = 'Alive' THEN 0 WHEN hospitaldischargestatus = 'Expired' THEN 1 ELSE NULL END` | 院内死亡数值化（0=存活，1=死亡） |
| `icu_los_hours` | `ROUND(unitdischargeoffset/60)` | ICU 住院时长（小时） |
| `hospital_los_hours` | `ROUND(hospitaldischargeoffset/60)` | 医院住院时长（小时） |

⚠️ **注意**：`icustay_detail.sql` 中的 gender 编码不同（0=女，1=男），请务必统一编码方式！

---

## SQL 模板

### 1. 提取基本人口统计学信息（基于 eicu-code）

```sql
-- 基于 eicu-code 官方实现：D:\Project\eicu-code\concepts\basic_demographics.sql
SELECT 
    pt.patientunitstayid,
    pt.age,
    pt.apacheadmissiondx,
    CASE 
        WHEN pt.gender = 'Male' THEN 1
        WHEN pt.gender = 'Female' THEN 2
        ELSE NULL 
    END AS gender_numeric,
    CASE 
        WHEN pt.hospitaldischargestatus = 'Alive' THEN 0
        WHEN pt.hospitaldischargestatus = 'Expired' THEN 1
        ELSE NULL 
    END AS hosp_mortality_numeric,
    ROUND(pt.unitdischargeoffset/60) AS icu_los_hours
FROM patient pt
ORDER BY pt.patientunitstayid;
```

### 2. 提取 ICU 住院详细信息（基于 eicu-code）

```sql
-- 基于 eicu-code 官方实现：D:\Project\eicu-code\concepts\icustay_detail.sql
SELECT 
    pt.uniquepid,
    pt.patienthealthsystemstayid,
    pt.patientunitstayid,
    pt.unitvisitnumber,
    pt.hospitalid,
    h.region,
    pt.unittype,
    pt.hospitaladmitoffset,
    pt.hospitaldischargeoffset,
    0 AS unitadmitoffset,  -- 相对于 ICU 入住的时间偏移（0 = 入住时刻）
    pt.unitdischargeoffset,
    ap.apachescore AS apache_iv,
    pt.hospitaldischargeyear,
    pt.age,
    CASE 
        WHEN LOWER(pt.gender) LIKE '%female%' THEN 0
        WHEN LOWER(pt.gender) LIKE '%male%' THEN 1
        ELSE NULL 
    END AS gender_numeric,  -- 注意：此处编码与 basic_demographics.sql 不同！
    pt.ethnicity,
    pt.admissionheight,
    pt.admissionweight,
    pt.dischargeweight,
    ROUND(pt.unitdischargeoffset/60) AS icu_los_hours
FROM patient pt
LEFT JOIN hospital h
    ON pt.hospitalid = h.hospitalid
LEFT JOIN apachepatientresult ap
    ON pt.patientunitstayid = ap.patientunitstayid
    AND ap.apacheversion = 'IV'
ORDER BY pt.uniquepid, pt.unitvisitnumber, pt.age;
```

⚠️ **重要提醒**：`icustay_detail.sql` 中的 gender 编码与 `basic_demographics.sql` 不同：
- `basic_demographics.sql`: 1=Male, 2=Female
- `icustay_detail.sql`: 0=Female, 1=Male

**建议**：统一使用 `basic_demographics.sql` 的编码（1=Male, 2=Female），因为更直观。

### 3. 筛选特定人群（如年龄 ≥ 18 岁）

```sql
-- 筛选成年患者（年龄 ≥ 18 岁）
SELECT 
    pt.patientunitstayid,
    pt.age,
    pt.gender,
    pt.ethnicity,
    pt.admissionweight,
    ROUND(pt.unitdischargeoffset/60) AS icu_los_hours,
    pt.hospitaldischargestatus
FROM patient pt
WHERE pt.age >= '18'  -- 注意：age 是 VARCHAR，但可以比较
  AND pt.age != '> 89'  -- 排除年龄 > 89 的患者（如果需要）
ORDER BY pt.patientunitstayid;
```

⚠️ **注意**：`age` 字段是 VARCHAR 类型，可能包含 `'> 89'` 这样的字符串。如果需要精确筛选，建议使用：

```sql
-- 更精确的年龄筛选（处理 '> 89' 的情况）
SELECT 
    pt.patientunitstayid,
    pt.age,
    pt.gender,
    CASE 
        WHEN pt.age = '> 89' THEN 90  -- 将 '> 89' 视为 90 岁
        ELSE pt.age::NUMERIC 
    END AS age_numeric
FROM patient pt
WHERE (
    pt.age = '> 89' 
    OR pt.age::NUMERIC >= 18
)
ORDER BY pt.patientunitstayid;
```

### 4. 统计 ICU 死亡率

```sql
-- 统计 ICU 死亡率（基于 eicu-code）
SELECT 
    COUNT(*) AS total_patients,
    SUM(CASE WHEN pt.hospitaldischargestatus = 'Expired' THEN 1 ELSE 0 END) AS deaths,
    ROUND(
        SUM(CASE WHEN pt.hospitaldischargestatus = 'Expired' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 
        2
    ) AS mortality_rate_percent
FROM patient pt;
```

### 5. 按 ICU 类型统计患者数

```sql
-- 按 ICU 类型统计患者数和死亡率
SELECT 
    pt.unittype,
    COUNT(*) AS total_patients,
    SUM(CASE WHEN pt.hospitaldischargestatus = 'Expired' THEN 1 ELSE 0 END) AS deaths,
    ROUND(
        SUM(CASE WHEN pt.hospitaldischargestatus = 'Expired' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 
        2
    ) AS mortality_rate_percent,
    ROUND(AVG(pt.unitdischargeoffset/60), 2) AS avg_icu_los_hours
FROM patient pt
WHERE pt.unittype IS NOT NULL
GROUP BY pt.unittype
ORDER BY total_patients DESC;
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

def get_basic_demographics(patient_ids=None):
    """
    提取基本人口统计学信息（基于 eicu-code）
    
    参考: D:\Project\eicu-code\concepts\basic_demographics.sql
    
    Parameters:
    -----------
    patient_ids : list or None
        患者 ID 列表，None 表示所有患者
    
    Returns:
    --------
    df : pandas DataFrame
        人口统计学信息
    """
    query = """
        SELECT 
            pt.patientunitstayid,
            pt.age,
            pt.apacheadmissiondx,
            CASE 
                WHEN pt.gender = 'Male' THEN 1
                WHEN pt.gender = 'Female' THEN 2
                ELSE NULL 
            END AS gender_numeric,
            CASE 
                WHEN pt.hospitaldischargestatus = 'Alive' THEN 0
                WHEN pt.hospitaldischargestatus = 'Expired' THEN 1
                ELSE NULL 
            END AS hosp_mortality_numeric,
            ROUND(pt.unitdischargeoffset/60) AS icu_los_hours
        FROM patient pt
        ORDER BY pt.patientunitstayid
    """
    
    # 添加患者 ID 过滤
    if patient_ids is not None:
        if len(patient_ids) == 1:
            query = query.replace(
                "ORDER BY pt.patientunitstayid",
                f"WHERE pt.patientunitstayid = {patient_ids[0]} ORDER BY pt.patientunitstayid"
            )
        else:
            ids_str = ",".join(map(str, patient_ids))
            query = f"""
                WITH target_patients AS (
                    SELECT unnest(ARRAY[{ids_str}]) AS pid
                )
                SELECT 
                    pt.patientunitstayid,
                    pt.age,
                    pt.apacheadmissiondx,
                    CASE 
                        WHEN pt.gender = 'Male' THEN 1
                        WHEN pt.gender = 'Female' THEN 2
                        ELSE NULL 
                    END AS gender_numeric,
                    CASE 
                        WHEN pt.hospitaldischargestatus = 'Alive' THEN 0
                        WHEN pt.hospitaldischargestatus = 'Expired' THEN 1
                        ELSE NULL 
                    END AS hosp_mortality_numeric,
                    ROUND(pt.unitdischargeoffset/60) AS icu_los_hours
                FROM patient pt, target_patients tp
                WHERE pt.patientunitstayid = tp.pid
                ORDER BY pt.patientunitstayid
            """
    
    df = pd.read_sql_query(query, conn)
    
    # 处理 age 字段（可能是 '> 89'）
    df['age_numeric'] = df['age'].apply(
        lambda x: 90 if x == '> 89' else (float(x) if x.replace('.', '').isdigit() else np.nan)
    )
    
    return df

def get_icu_stay_detail(patient_ids=None):
    """
    提取 ICU 住院详细信息（基于 eicu-code）
    
    参考: D:\Project\eicu-code\concepts\icustay_detail.sql
    
    Parameters:
    -----------
    patient_ids : list or None
        患者 ID 列表，None 表示所有患者
    
    Returns:
    --------
    df : pandas DataFrame
        ICU 住院详细信息
    """
    query = """
        SELECT 
            pt.uniquepid,
            pt.patienthealthsystemstayid,
            pt.patientunitstayid,
            pt.unitvisitnumber,
            pt.hospitalid,
            h.region,
            pt.unittype,
            pt.hospitaladmitoffset,
            pt.hospitaldischargeoffset,
            0 AS unitadmitoffset,
            pt.unitdischargeoffset,
            ap.apachescore AS apache_iv,
            pt.hospitaldischargeyear,
            pt.age,
            CASE 
                WHEN LOWER(pt.gender) LIKE '%female%' THEN 0
                WHEN LOWER(pt.gender) LIKE '%male%' THEN 1
                ELSE NULL 
            END AS gender_numeric,  -- 注意：此处编码与 basic_demographics.sql 不同！
            pt.ethnicity,
            pt.admissionheight,
            pt.admissionweight,
            pt.dischargeweight,
            ROUND(pt.unitdischargeoffset/60) AS icu_los_hours
        FROM patient pt
        LEFT JOIN hospital h
            ON pt.hospitalid = h.hospitalid
        LEFT JOIN apachepatientresult ap
            ON pt.patientunitstayid = ap.patientunitstayid
            AND ap.apacheversion = 'IV'
        ORDER BY pt.uniquepid, pt.unitvisitnumber, pt.age
    """
    
    # 添加患者 ID 过滤
    if patient_ids is not None:
        if len(patient_ids) == 1:
            query = query.replace(
                "ORDER BY pt.uniquepid",
                f"WHERE pt.patientunitstayid = {patient_ids[0]} ORDER BY pt.uniquepid"
            )
        else:
            ids_str = ",".join(map(str, patient_ids))
            query = f"""
                WITH target_patients AS (
                    SELECT unnest(ARRAY[{ids_str}]) AS pid
                )
                SELECT 
                    pt.uniquepid,
                    pt.patienthealthsystemstayid,
                    pt.patientunitstayid,
                    pt.unitvisitnumber,
                    pt.hospitalid,
                    h.region,
                    pt.unittype,
                    pt.hospitaladmitoffset,
                    pt.hospitaldischargeoffset,
                    0 AS unitadmitoffset,
                    pt.unitdischargeoffset,
                    ap.apachescore AS apache_iv,
                    pt.hospitaldischargeyear,
                    pt.age,
                    CASE 
                        WHEN LOWER(pt.gender) LIKE '%female%' THEN 0
                        WHEN LOWER(pt.gender) LIKE '%male%' THEN 1
                        ELSE NULL 
                    END AS gender_numeric,
                    pt.ethnicity,
                    pt.admissionheight,
                    pt.admissionweight,
                    pt.dischargeweight,
                    ROUND(pt.unitdischargeoffset/60) AS icu_los_hours
                FROM patient pt
                LEFT JOIN hospital h
                    ON pt.hospitalid = h.hospitalid
                LEFT JOIN apachepatientresult ap
                    ON pt.patientunitstayid = ap.patientunitstayid
                    AND ap.apacheversion = 'IV'
                WHERE pt.patientunitstayid IN ({ids_str})
                ORDER BY pt.uniquepid, pt.unitvisitnumber, pt.age
            """
    
    df = pd.read_sql_query(query, conn)
    
    # 处理 age 字段（可能是 '> 89'）
    df['age_numeric'] = df['age'].apply(
        lambda x: 90 if x == '> 89' else (float(x) if x.replace('.', '').isdigit() else np.nan)
    )
    
    return df

def calculate_mortality_rate(df, status_col='hospitaldischargestatus'):
    """
    计算死亡率
    
    Parameters:
    -----------
    df : pandas DataFrame
        包含 hospitaldischargestatus 列的 DataFrame
    status_col : str
        出院状态列名
    
    Returns:
    --------
    mortality_rate : float
        死亡率（0-1 之间）
    """
    total = len(df)
    deaths = len(df[df[status_col] == 'Expired'])
    mortality_rate = deaths / total if total > 0 else np.nan
    return mortality_rate

# 使用示例
if __name__ == "__main__":
    # 示例 1：提取所有患者的基本人口统计学信息
    demo = get_basic_demographics()
    print(f"提取了 {len(demo)} 条记录")
    print(demo.head(10))
    
    # 示例 2：提取特定患者的 ICU 住院详细信息
    detail = get_icu_stay_detail(patient_ids=[12345, 67890])
    print(f"\n提取了 {len(detail)} 条记录")
    print(detail.head(10))
    
    # 示例 3：计算总体死亡率
    mortality_rate = calculate_mortality_rate(demo, status_col='hosp_mortality_numeric')
    print(f"\n总体死亡率: {mortality_rate:.2%}")
    
    conn.close()
```

---

## 注意事项

### 1. 字段名规范

✅ **正确** (小写):
```sql
SELECT pt.patientunitstayid, pt.age, pt.gender
FROM patient pt
WHERE pt.age >= '18'
```

❌ **错误** (camelCase):
```sql
SELECT pt.patientUnitStayID, pt.Age, pt.Gender  -- 字段不存在！
FROM patient pt
WHERE pt.Age >= 18
```

### 2. 数据类型转换

- `age` 字段类型为 `VARCHAR`，可能包含 `'> 89'` 这样的字符串
- 计算时需转换：`CASE WHEN age = '> 89' THEN 90 ELSE age::NUMERIC END`
- `unitdischargeoffset` 单位为分钟，转换为小时：`ROUND(unitdischargeoffset/60)`

### 3. 时间偏移量

- `hospitaladmitoffset`: 医院入院时间偏移（分钟，通常是负数）
- `hospitaldischargeoffset`: 医院出院时间偏移（分钟）
- `unitdischargeoffset`: ICU 出院时间偏移（分钟，0 = ICU 入住时刻）
- ICU 住院时长：`unitdischargeoffset`（分钟）或 `ROUND(unitdischargeoffset/60)`（小时）

### 4. 数据质量

- **年龄**: 可能包含 `'> 89'`，需要特殊处理
- **身高/体重**: 可能缺失，查询时考虑使用 `IS NOT NULL` 过滤
- **种族**: 可能缺失或不标准，建议进行标准化

### 5. gender 编码不一致（重要！）

⚠️ **注意**：eicu-code 中的两个文件使用了不同的 gender 编码：

| 文件 | Male | Female |
|------|------|--------|
| `basic_demographics.sql` | 1 | 2 |
| `icustay_detail.sql` | 1 | 0 |

**建议**：统一使用 `basic_demographics.sql` 的编码（1=Male, 2=Female），因为更直观。

### 6. 性能优化

- `patient` 表有 **约 20 万条记录**（eICU 2.0）
- 查询时务必:
  - 使用 `patientunitstayid` 过滤
  - 添加 `LIMIT` 子句（测试时）
  - 使用 `EXPLAIN` 分析查询计划

---

## 常用查询场景

### 1. 提取存活患者的信息

```sql
SELECT 
    pt.patientunitstayid,
    pt.age,
    pt.gender,
    pt.admissionweight,
    ROUND(pt.unitdischargeoffset/60) AS icu_los_hours
FROM patient pt
WHERE pt.hospitaldischargestatus = 'Alive'
ORDER BY pt.patientunitstayid
LIMIT 100;
```

### 2. 提取死亡患者的信息

```sql
SELECT 
    pt.patientunitstayid,
    pt.age,
    pt.gender,
    pt.admissionweight,
    ROUND(pt.unitdischargeoffset/60) AS icu_los_hours,
    pt.apacheadmissiondx
FROM patient pt
WHERE pt.hospitaldischargestatus = 'Expired'
ORDER BY pt.patientunitstayid
LIMIT 100;
```

### 3. 按年龄组统计死亡率

```sql
-- 按年龄组统计死亡率
WITH age_groups AS (
    SELECT 
        pt.patientunitstayid,
        pt.hospitaldischargestatus,
        CASE 
            WHEN pt.age = '> 89' THEN '> 89'
            WHEN pt.age::NUMERIC < 40 THEN '< 40'
            WHEN pt.age::NUMERIC BETWEEN 40 AND 59 THEN '40-59'
            WHEN pt.age::NUMERIC BETWEEN 60 AND 79 THEN '60-79'
            ELSE '≥ 80'
        END AS age_group
    FROM patient pt
    WHERE pt.age != '> 89' OR pt.age = '> 89'
)
SELECT 
    age_group,
    COUNT(*) AS total_patients,
    SUM(CASE WHEN hospitaldischargestatus = 'Expired' THEN 1 ELSE 0 END) AS deaths,
    ROUND(
        SUM(CASE WHEN hospitaldischargestatus = 'Expired' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 
        2
    ) AS mortality_rate_percent
FROM age_groups
GROUP BY age_group
ORDER BY age_group;
```

---

## 参考链接

- **patient 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/patient.html
- **hospital 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/hospital.html
- **apachepatientresult 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/apachepatientresult.html
- **eICU 数据介绍**: https://eicu-crd.mit.edu/

---

**最后更新**: 2026-05-22  
**更新人**: 悟空（基于 eicu-code 官方实现修正）
