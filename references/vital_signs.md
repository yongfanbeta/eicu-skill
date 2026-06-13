# eICU 2.0 生命体征查询

## ⚠️ 重要提示：两种提取生命体征的方法

eICU 2.0 提供**两种**提取生命体征的方法：

| 方法 | 数据来源 | 推荐使用场景 |
|------|----------|--------------|
| **方法 1**（本文档） | `vitalperiodic` + `vitalaperiodic` 表 | 需要高频数据（如心率变异性分析） |
| **方法 2**（`vital_signs_nursecharting.md`） | `nursecharting` 表（基于 eicu-code） | 需要可靠的生命体征（如计算 SOFA 评分） |

**建议**: 
- 如果需要**高频数据**（如心率变异性分析），使用本文档（方法 1）
- 如果需要**可靠的生命体征**（如计算 SOFA 评分），使用 [`vital_signs_nursecharting.md`](./vital_signs_nursecharting.md)（方法 2）

---

## 目录

1. [指标映射](#指标映射)
2. [SQL 模板：ICU 第一天生命体征](#sql-模板icu-第一天生命体征)
3. [Python 模板](#python-模板)
4. [注意事项](#注意事项)

---

## 指标映射

eICU 生命体征分两个表：`vitalperiodic`（周期性，每 1-5 分钟记录）和 `vitalaperiodic`（非周期性，有创监测）。

**重要**: eICU **没有** `vitalSignDerived` 表，这是错误的表名。

| 指标 | 表名 | 字段名 | 类型 | 单位 | 说明 |
|------|------|--------|------|------|------|
| 心率 | vitalperiodic | heartrate | int4 | bpm | 心率 |
| 体温 | vitalperiodic | temperature | numeric(11,4) | °C | 体温（摄氏）|
| 血氧饱和度 | vitalperiodic | sao2 | int4 | % | 血氧饱和度 |
| 呼吸频率 | vitalperiodic | respiration | int4 | breaths/min | 呼吸频率 |
| 有创收缩压 | vitalperiodic | systemicsystolic | int4 | mmHg | 有创收缩压 |
| 有创舒张压 | vitalperiodic | systemicdiastolic | int4 | mmHg | 有创舒张压 |
| 有创平均动脉压 | vitalperiodic | systemicmean | int4 | mmHg | 有创平均动脉压 |
| 肺动脉收缩压 | vitalperiodic | pasystolic | int4 | mmHg | 肺动脉收缩压 |
| 肺动脉舒张压 | vitalperiodic | padiastolic | int4 | mmHg | 肺动脉舒张压 |
| 肺动脉平均压 | vitalperiodic | pamean | int4 | mmHg | 肺动脉平均压 |
| 中心静脉压 | vitalperiodic | cvp | int4 | mmHg | 中心静脉压 |
| 呼气末 CO2 | vitalperiodic | etco2 | int4 | mmHg | 呼气末 CO2 |
| ST 段导联 1 | vitalperiodic | st1 | float8 | mV | ST 段偏移 |
| ST 段导联 2 | vitalperiodic | st2 | float8 | mV | ST 段偏移 |
| ST 段导联 3 | vitalperiodic | st3 | float8 | mV | ST 段偏移 |
| 颅内压 | vitalperiodic | icp | int4 | mmHg | 颅内压 |
| 无创收缩压 | vitalaperiodic | noninvasivesystolic | float8 | mmHg | 无创收缩压 |
| 无创舒张压 | vitalaperiodic | noninvasivediastolic | float8 | mmHg | 无创舒张压 |
| 无创平均动脉压 | vitalaperiodic | noninvasivemean | float8 | mmHg | 无创平均动脉压 |
| 肺动脉闭塞压 | vitalaperiodic | paop | float8 | mmHg | PCWP |
| 心输出量 | vitalaperiodic | cardiacoutput | float8 | L/min | 心输出量 |
| 心脏指数 | vitalaperiodic | cardiacinput | float8 | L/min/m² | 心脏指数 |
| 全身血管阻力 | vitalaperiodic | svr | float8 | dyn·s·cm⁻⁵ | SVR |
| 全身血管阻力指数 | vitalaperiodic | svri | float8 | dyn·s·cm⁻⁵/m² | SVRI |
| 肺血管阻力 | vitalaperiodic | pvr | float8 | dyn·s·cm⁻⁵ | PVR |
| 肺血管阻力指数 | vitalaperiodic | pvri | float8 | dyn·s·cm⁻⁵/m² | PVRI |

**数据分布**:
- `vitalperiodic`: 146,671,642 条记录（最常用）
- `vitalaperiodic`: 25,075,074 条记录（有创监测，数据较少）

---

## SQL 模板：ICU 第一天生命体征

ICU 第一天 = `observationoffset` 0 到 1440 分钟。

### 1. 长格式（推荐）：提取多个生命体征

```sql
-- 提取 ICU 第一天多个生命体征（长格式）
SELECT
    vp.patientunitstayid,
    'heartrate' AS vital_name,
    vp.observationoffset,
    vp.heartrate AS valuenum
FROM vitalperiodic vp
WHERE vp.heartrate IS NOT NULL
  AND vp.heartrate > 0 AND vp.heartrate < 350
  AND vp.observationoffset >= 0 AND vp.observationoffset < 1440

UNION ALL

SELECT
    vp.patientunitstayid,
    'temperature' AS vital_name,
    vp.observationoffset,
    vp.temperature AS valuenum
FROM vitalperiodic vp
WHERE vp.temperature IS NOT NULL
  AND vp.temperature > 20 AND vp.temperature < 45
  AND vp.observationoffset >= 0 AND vp.observationoffset < 1440

UNION ALL

SELECT
    vp.patientunitstayid,
    'sao2' AS vital_name,
    vp.observationoffset,
    vp.sao2::numeric AS valuenum
FROM vitalperiodic vp
WHERE vp.sao2 IS NOT NULL
  AND vp.sao2 > 0 AND vp.sao2 <= 100
  AND vp.observationoffset >= 0 AND vp.observationoffset < 1440

UNION ALL

SELECT
    vp.patientunitstayid,
    'respiration' AS vital_name,
    vp.observationoffset,
    vp.respiration::numeric AS valuenum
FROM vitalperiodic vp
WHERE vp.respiration IS NOT NULL
  AND vp.respiration > 0 AND vp.respiration < 60
  AND vp.observationoffset >= 0 AND vp.observationoffset < 1440

UNION ALL

SELECT
    vp.patientunitstayid,
    'systemicsystolic' AS vital_name,
    vp.observationoffset,
    vp.systemicsystolic::numeric AS valuenum
FROM vitalperiodic vp
WHERE vp.systemicsystolic IS NOT NULL
  AND vp.systemicsystolic > 0 AND vp.systemicsystolic < 300
  AND vp.observationoffset >= 0 AND vp.observationoffset < 1440

UNION ALL

SELECT
    vp.patientunitstayid,
    'systemicdiastolic' AS vital_name,
    vp.observationoffset,
    vp.systemicdiastolic::numeric AS valuenum
FROM vitalperiodic vp
WHERE vp.systemicdiastolic IS NOT NULL
  AND vp.systemicdiastolic > 0 AND vp.systemicdiastolic < 300
  AND vp.observationoffset >= 0 AND vp.observationoffset < 1440

UNION ALL

SELECT
    vp.patientunitstayid,
    'systemicmean' AS vital_name,
    vp.observationoffset,
    vp.systemicmean::numeric AS valuenum
FROM vitalperiodic vp
WHERE vp.systemicmean IS NOT NULL
  AND vp.systemicmean > 0 AND vp.systemicmean < 200
  AND vp.observationoffset >= 0 AND vp.observationoffset < 1440

UNION ALL

SELECT
    va.patientunitstayid,
    'noninvasivesystolic' AS vital_name,
    va.observationoffset,
    va.noninvasivesystolic AS valuenum
FROM vitalaperiodic va
WHERE va.noninvasivesystolic IS NOT NULL
  AND va.noninvasivesystolic > 0 AND va.noninvasivesystolic < 300
  AND va.observationoffset >= 0 AND va.observationoffset < 1440

ORDER BY patientunitstayid, vital_name, observationoffset;
```

### 2. 宽表格式：每个患者一行

```sql
-- 提取 ICU 第一天生命体征（宽表格式）
SELECT
    p.patientunitstayid,
    p.age,
    p.gender,
    p.admissionweight,
    -- 心率
    AVG(CASE WHEN vp.heartrate IS NOT NULL 
             AND vp.heartrate > 0 AND vp.heartrate < 350 
             THEN vp.heartrate END) AS heartrate_mean,
    MIN(CASE WHEN vp.heartrate IS NOT NULL 
             AND vp.heartrate > 0 AND vp.heartrate < 350 
             THEN vp.heartrate END) AS heartrate_min,
    MAX(CASE WHEN vp.heartrate IS NOT NULL 
             AND vp.heartrate > 0 AND vp.heartrate < 350 
             THEN vp.heartrate END) AS heartrate_max,
    -- 体温
    AVG(CASE WHEN vp.temperature IS NOT NULL 
             AND vp.temperature > 20 AND vp.temperature < 45 
             THEN vp.temperature END) AS temperature_mean,
    -- 血氧饱和度
    AVG(CASE WHEN vp.sao2 IS NOT NULL 
             AND vp.sao2 > 0 AND vp.sao2 <= 100 
             THEN vp.sao2 END) AS sao2_mean,
    -- 呼吸频率
    AVG(CASE WHEN vp.respiration IS NOT NULL 
             AND vp.respiration > 0 AND vp.respiration < 60 
             THEN vp.respiration END) AS respiration_mean,
    -- 有创血压
    AVG(CASE WHEN vp.systemicsystolic IS NOT NULL 
             AND vp.systemicsystolic > 0 AND vp.systemicsystolic < 300 
             THEN vp.systemicsystolic END) AS sbp_invasive_mean,
    AVG(CASE WHEN vp.systemicdiastolic IS NOT NULL 
             AND vp.systemicdiastolic > 0 AND vp.systemicdiastolic < 300 
             THEN vp.systemicdiastolic END) AS dbp_invasive_mean,
    AVG(CASE WHEN vp.systemicmean IS NOT NULL 
             AND vp.systemicmean > 0 AND vp.systemicmean < 200 
             THEN vp.systemicmean END) AS map_invasive_mean,
    -- 中心静脉压
    AVG(CASE WHEN vp.cvp IS NOT NULL 
             AND vp.cvp > 0 AND vp.cvp < 50 
             THEN vp.cvp END) AS cvp_mean
FROM patient p
LEFT JOIN vitalperiodic vp
    ON p.patientunitstayid = vp.patientunitstayid
    AND vp.observationoffset >= 0 AND vp.observationoffset < 1440
GROUP BY p.patientunitstayid, p.age, p.gender, p.admissionweight
ORDER BY p.patientunitstayid;
```

### 3. 提取第一次记录的生命体征

```sql
-- 每个患者 ICU 第一天的第一次生命体征记录
WITH first_vitals AS (
    SELECT
        vp.patientunitstayid,
        vp.observationoffset,
        vp.heartrate,
        vp.temperature,
        vp.sao2,
        vp.respiration,
        vp.systemicsystolic,
        vp.systemicdiastolic,
        vp.systemicmean,
        vp.cvp,
        ROW_NUMBER() OVER (
            PARTITION BY vp.patientunitstayid 
            ORDER BY vp.observationoffset
        ) AS rn
    FROM vitalperiodic vp
    WHERE vp.observationoffset >= 0 AND vp.observationoffset < 1440
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

# 数据库连接
conn = psycopg2.connect(
    host="your_host",
    port=5432,
    dbname="eicu",
    user="your_user",
    password="your_password"
)

def extract_first_day_vitals(patient_ids=None, vitals_list=None):
    """
    提取 ICU 第一天（0-1440 分钟）的生命体征
    
    Parameters:
    -----------
    patient_ids : list or None
        患者 ID 列表，None 表示所有患者
    vitals_list : list or None
        要提取的生命体征列表，None 表示全部
        可选值: 'heartrate', 'temperature', 'sao2', 'respiration',
               'systemicsystolic', 'systemicdiastolic', 'systemicmean',
               'cvp', 'noninvasivesystolic', etc.
    
    Returns:
    --------
    df : pandas DataFrame
        长格式生命体征数据
    """
    if vitals_list is None:
        vitals_list = ['heartrate', 'temperature', 'sao2', 'respiration',
                      'systemicsystolic', 'systemicdiastolic', 'systemicmean']
    
    # 构建 UNION ALL 查询
    queries = []
    
    # vitalperiodic 表中的指标
    vitalperiodic_fields = {
        'heartrate': 'heartrate',
        'temperature': 'temperature',
        'sao2': 'sao2',
        'respiration': 'respiration',
        'systemicsystolic': 'systemicsystolic',
        'systemicdiastolic': 'systemicdiastolic',
        'systemicmean': 'systemicmean',
        'cvp': 'cvp',
        'etco2': 'etco2'
    }
    
    for vital_name in vitals_list:
        if vital_name in vitalperiodic_fields:
            field = vitalperiodic_fields[vital_name]
            q = f"""
                SELECT 
                    vp.patientunitstayid,
                    '{vital_name}' AS vital_name,
                    vp.observationoffset,
                    vp.{field} AS valuenum
                FROM vitalperiodic vp
                WHERE vp.{field} IS NOT NULL
                  AND vp.observationoffset >= 0 
                  AND vp.observationoffset < 1440
            """
            queries.append(q)
    
    # vitalaperiodic 表中的指标
    vitalaperiodic_fields = {
        'noninvasivesystolic': 'noninvasivesystolic',
        'noninvasivediastolic': 'noninvasivediastolic',
        'noninvasivemean': 'noninvasivemean',
        'paop': 'paop',
        'cardiacoutput': 'cardiacoutput'
    }
    
    for vital_name in vitals_list:
        if vital_name in vitalaperiodic_fields:
            field = vitalaperiodic_fields[vital_name]
            q = f"""
                SELECT 
                    va.patientunitstayid,
                    '{vital_name}' AS vital_name,
                    va.observationoffset,
                    va.{field} AS valuenum
                FROM vitalaperiodic va
                WHERE va.{field} IS NOT NULL
                  AND va.observationoffset >= 0 
                  AND va.observationoffset < 1440
            """
            queries.append(q)
    
    # 合并查询
    full_query = " UNION ALL ".join(queries) + " ORDER BY patientunitstayid, vital_name, observationoffset"
    
    # 添加患者 ID 过滤
    if patient_ids is not None:
        if len(patient_ids) == 1:
            full_query = full_query.replace(
                "WHERE ", 
                f"WHERE vp.patientunitstayid = {patient_ids[0]} AND "
            ).replace(
                "WHERE va.", 
                f"WHERE va.patientunitstayid = {patient_ids[0]} AND "
            )
        else:
            # 使用临时表或 ANY 语法
            full_query = """
                WITH target_patients AS (
                    SELECT unnest(%(patient_ids)s) AS pid
                )
            """ + full_query.replace(
                "FROM vitalperiodic vp",
                "FROM vitalperiodic vp, target_patients tp WHERE vp.patientunitstayid = tp.pid"
            ).replace(
                "FROM vitalaperiodic va",
                "FROM vitalaperiodic va, target_patients tp WHERE va.patientunitstayid = tp.pid"
            )
    
    # 执行查询
    if patient_ids is not None and len(patient_ids) > 1:
        df = pd.read_sql_query(full_query, conn, params={"patient_ids": patient_ids})
    else:
        df = pd.read_sql_query(full_query, conn)
    
    return df

def aggregate_vitals_long_to_wide(df):
    """
    将长格式生命体征数据转换为宽表格式
    
    Parameters:
    -----------
    df : pandas DataFrame
        长格式数据（来自 extract_first_day_vitals）
    
    Returns:
    --------
    wide_df : pandas DataFrame
        宽表格式，每个患者一行
    """
    # 计算统计量
    agg_df = df.groupby(['patientunitstayid', 'vital_name']).agg(
        mean_value=('valuenum', 'mean'),
        min_value=('valuenum', 'min'),
        max_value=('valuenum', 'max'),
        count=('valuenum', 'count')
    ).reset_index()
    
    # 转换为宽表
    wide_df = agg_df.pivot(
        index='patientunitstayid',
        columns='vital_name',
        values=['mean_value', 'min_value', 'max_value', 'count']
    )
    
    # 展平列名
    wide_df.columns = [f"{col[1]}_{col[0]}" for col in wide_df.columns]
    wide_df = wide_df.reset_index()
    
    return wide_df

# 使用示例
if __name__ == "__main__":
    # 示例 1：提取所有患者的心率和体温
    vitals = extract_first_day_vitals(
        vitals_list=['heartrate', 'temperature']
    )
    print(f"提取了 {len(vitals)} 条记录")
    print(vitals.head(10))
    
    # 示例 2：提取特定患者的所有生命体征
    vitals = extract_first_day_vitals(
        patient_ids=[154347, 112001, 189456]
    )
    
    # 示例 3：转换为宽表
    wide_df = aggregate_vitals_long_to_wide(vitals)
    print(wide_df.head(5))
    
    conn.close()
```

---

## 注意事项

### 1. 表选择

- **vitalperiodic**: 最常用，记录周期性测量的生命体征（每 1-5 分钟）
- **vitalaperiodic**: 记录有创监测数据（如 PACWP、心输出量），数据较少但更重要

### 2. 字段名规范

✅ **正确** (小写):
```sql
SELECT vp.heartrate, vp.observationoffset
FROM vitalperiodic vp
```

❌ **错误** (camelCase):
```sql
SELECT vp.heartRate, vp.observationOffset  -- 字段不存在！
FROM vitalPeriodic vp
```

### 3. 数据质量

- **缺失值多**: 并非每个时间点都有所有指标
- **异常值**: 查询时务必加合理的范围过滤（如 `heartrate > 0 AND heartrate < 350`）
- **单位**: 
  - 体温为摄氏度（°C）
  - 血压为 mmHg
  - 心率单位为 bpm

### 4. 性能优化

- `vitalperiodic` 表有 **1.46 亿条记录**，查询时务必：
  - 加 `LIMIT` 子句（测试时）
  - 使用 `patientunitstayid` 过滤（不要全表扫描）
  - 限定 `observationoffset` 范围
- 使用 `EXPLAIN` 分析查询计划

### 5. 时间表示

- `observationoffset` 是相对于 **ICU 入住时间** 的分钟数
- `observationoffset = 0`: ICU 入住时刻
- `observationoffset = 1440`: 入住后 24 小时
- 负值表示入住前的记录（如有）

### 6. 有创 vs 无创血压

- **有创血压** (`systemicsystolic`, `systemicdiastolic`, `systemicmean`): 来自动脉导管，更准确
- **无创血压** (`noninvasivesystolic`, `noninvasivediastolic`, `noninvasivemean`): 来自袖带测量，可能不准确

**建议**: 优先使用有创血压，无创血压作为补充。

---

## 常用查询场景

### 1. 提取 ICU 前 6 小时的生命体征

```sql
SELECT patientunitstayid, vital_name, observationoffset, valuenum
FROM (
    -- 同上，使用 UNION ALL
) all_vitals
WHERE observationoffset >= 0 AND observationoffset < 360  -- 前 6 小时
ORDER BY patientunitstayid, vital_name, observationoffset;
```

### 2. 提取最异常的生命体征（如最高/最低值）

```sql
SELECT 
    patientunitstayid,
    MAX(heartrate) AS max_heartrate,
    MIN(heartrate) AS min_heartrate,
    MAX(sao2) AS max_sao2,
    MIN(sao2) AS min_sao2
FROM (
    SELECT patientunitstayid, heartrate, sao2
    FROM vitalperiodic
    WHERE observationoffset BETWEEN 0 AND 1440
) vp
GROUP BY patientunitstayid;
```

### 3. 计算生命体征的变异度（标准差）

```sql
SELECT 
    patientunitstayid,
    STDDEV(heartrate) AS hr_stddev,
    COEFF_VAR(heartrate) AS hr_cv  -- 变异系数
FROM vitalperiodic
WHERE observationoffset BETWEEN 0 AND 1440
  AND heartrate IS NOT NULL
GROUP BY patientunitstayid;
```

---

## 参考链接

- **vitalperiodic 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/vitalperiodic.html
- **vitalaperiodic 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/vitalaperiodic.html

---

**最后更新**: 2026-05-20  
**更新人**: 悟空（基于 MIT 官方 SchemaSpy 文档修正）
