# eICU 2.0 生化指标查询（基于 eicu-code 官方实现）

**参考文件**: `D:\Project\eicu-code\concepts\labsfirstday.sql`

---

## 目录

1. [常用 labname 映射（准确值）](#常用-labname-映射准确值)
2. [SQL 模板（精确匹配 + 异常值检查）](#sql-模板精确匹配--异常值检查)
3. [Python 模板（精确匹配 + 异常值检查）](#python-模板精确匹配--异常值检查)
4. [注意事项（基于 eicu-code）](#注意事项基于-eicu-code)

---

## 常用 labname 映射（准确值）

> **重要**: 基于 eicu-code 官方实现，**使用精确匹配**（`labname = 'xxx'`）而非模糊匹配（`ILIKE`）。

| 指标 | 准确 labname | 异常值检查 | 备注 |
|------|-------------|--------------|------|
| 阴离子间隙 | `'anion gap'` | > 10000 则视为 NULL | mEq/L |
| 白蛋白 | `'albumin'` | > 10 则视为 NULL | g/dL |
| bands | `'-bands'` | < 0 或 > 100 则视为 NULL | % |
| 碳酸氢根 | `'bicarbonate'` 或 `'HCO3'` | > 10000 则视为 NULL | mEq/L，两个都可能有 |
| 总胆红素 | `'total bilirubin'` | > 150 则视为 NULL | mg/dL |
| 肌酐 | `'creatinine'` | > 150 则视为 NULL | mg/dL |
| 氯 | `'chloride'` | > 10000 则视为 NULL | mEq/L |
| 葡萄糖 | `'glucose'` | > 10000 则视为 NULL | mg/dL |
| 红细胞压积 | `'Hct'` | > 100 则视为 NULL | % |
| 血红蛋白 | `'Hgb'` | > 50 则视为 NULL | g/dL |
| 乳酸 | `'lactate'` | > 50 则视为 NULL | mmol/L |
| 血小板 | `'platelets x 1000'` | > 10000 则视为 NULL | K/uL |
| 钾 | `'potassium'` | > 30 则视为 NULL | mEq/L |
| PTT | `'PTT'` | > 150 则视为 NULL | sec |
| INR | `'PT - INR'` | > 50 则视为 NULL |  |
| PT | `'PT'` | > 150 则视为 NULL | sec |
| 钠 | `'sodium'` | > 200 则视为 NULL | mEq/L |
| BUN | `'BUN'` | > 300 则视为 NULL |  |
| 白细胞 | `'WBC x 1000'` | > 1000 则视为 NULL | K/uL |

---

## SQL 模板（精确匹配 + 异常值检查）

### 1. 宽表格式：提取第一天所有实验室检查（基于 eicu-code）

```sql
-- 基于 eicu-code 官方实现：D:\Project\eicu-code\concepts\labsfirstday.sql
-- 特点：
--   1. 使用精确匹配（labname = 'xxx'）
--   2. 包含异常值检查（sanity checks）
--   3. 直接使用 labname 过滤（不需要 ILIKE）

SELECT
  pvt.patientunitstayid,
  min(CASE WHEN labname = 'anion gap' THEN labresult ELSE null END) as aniongap_min,
  max(CASE WHEN labname = 'anion gap' THEN labresult ELSE null END) as aniongap_max,
  min(CASE WHEN labname = 'albumin' THEN labresult ELSE null END) as albumin_min,
  max(CASE WHEN labname = 'albumin' THEN labresult ELSE null END) as albumin_max,
  min(CASE WHEN labname = '-bands' THEN labresult ELSE null END) as bands_min,
  max(CASE WHEN labname = '-bands' THEN labresult ELSE null END) as bands_max,
  min(CASE WHEN labname = 'bicarbonate' THEN labresult ELSE null END) as bicarbonate_min,
  max(CASE WHEN labname = 'bicarbonate' THEN labresult ELSE null END) as bicarbonate_max,
  min(CASE WHEN labname = 'HCO3' THEN labresult ELSE null END) as hco3_min,
  max(CASE WHEN labname = 'HCO3' THEN labresult ELSE null END) as hco3_max,
  min(CASE WHEN labname = 'total bilirubin' THEN labresult ELSE null END) as bilirubin_total_min,
  max(CASE WHEN labname = 'total bilirubin' THEN labresult ELSE null END) as bilirubin_total_max,
  min(CASE WHEN labname = 'creatinine' THEN labresult ELSE null END) as creatinine_min,
  max(CASE WHEN labname = 'creatinine' THEN labresult ELSE null END) as creatinine_max,
  min(CASE WHEN labname = 'chloride' THEN labresult ELSE null END) as chloride_min,
  max(CASE WHEN labname = 'chloride' THEN labresult ELSE null END) as chloride_max,
  min(CASE WHEN labname = 'glucose' THEN labresult ELSE null END) as glucose_min,
  max(CASE WHEN labname = 'glucose' THEN labresult ELSE null END) as glucose_max,
  min(CASE WHEN labname = 'Hct' THEN labresult ELSE null END) as hematocrit_min,
  max(CASE WHEN labname = 'Hct' THEN labresult ELSE null END) as hematocrit_max,
  min(CASE WHEN labname = 'Hgb' THEN labresult ELSE null END) as hemoglobin_min,
  max(CASE WHEN labname = 'Hgb' THEN labresult ELSE null END) as hemoglobin_max,
  min(CASE WHEN labname = 'lactate' THEN labresult ELSE null END) as lactate_min,
  max(CASE WHEN labname = 'lactate' THEN labresult ELSE null END) as lactate_max,
  min(CASE WHEN labname = 'platelets x 1000' THEN labresult ELSE null END) as platelets_min,
  max(CASE WHEN labname = 'platelets x 1000' THEN labresult ELSE null END) as platelets_max,
  min(CASE WHEN labname = 'potassium' THEN labresult ELSE null END) as potassium_min,
  max(CASE WHEN labname = 'potassium' THEN labresult ELSE null END) as potassium_max,
  min(CASE WHEN labname = 'PTT' THEN labresult ELSE null END) as ptt_min,
  max(CASE WHEN labname = 'PTT' THEN labresult ELSE null END) as ptt_max,
  min(CASE WHEN labname = 'PT - INR' THEN labresult ELSE null END) as inr_min,
  max(CASE WHEN labname = 'PT - INR' THEN labresult ELSE null END) as inr_max,
  min(CASE WHEN labname = 'PT' THEN labresult ELSE null END) as pt_min,
  max(CASE WHEN labname = 'PT' THEN labresult ELSE null END) as pt_max,
  min(CASE WHEN labname = 'sodium' THEN labresult ELSE null END) as sodium_min,
  max(CASE WHEN labname = 'sodium' THEN labresult ELSE null END) as sodium_max,
  min(CASE WHEN labname = 'BUN' THEN labresult ELSE null END) as bun_min,
  max(CASE WHEN labname = 'BUN' THEN labresult ELSE null END) as bun_max,
  min(CASE WHEN labname = 'WBC x 1000' THEN labresult ELSE null END) as wbc_min,
  max(CASE WHEN labname = 'WBC x 1000' THEN labresult ELSE null END) as wbc_max
FROM
( -- begin query that extracts the data
  SELECT p.patientunitstayid, le.labname
  , CASE
      WHEN labname = 'albumin' and le.labresult >    10 THEN null -- g/dL 'ALBUMIN'
      WHEN labname = 'anion gap' and le.labresult > 10000 THEN null -- mEq/L 'ANION GAP'
      WHEN labname = '-bands' and le.labresult <     0 THEN null -- immature band forms, %
      WHEN labname = '-bands' and le.labresult >   100 THEN null -- immature band forms, %
      WHEN labname = 'bicarbonate' and le.labresult > 10000 THEN null -- mEq/L 'BICARBONATE'
      WHEN labname = 'HCO3' and le.labresult > 10000 THEN null -- mEq/L 'BICARBONATE'
      WHEN labname = 'total bilirubin' and le.labresult >   150 THEN null -- mg/dL 'BILIRUBIN'
      WHEN labname = 'creatinine' and le.labresult >   150 THEN null -- mg/dL 'CREATININE'
      WHEN labname = 'chloride' and le.labresult > 10000 THEN null -- mEq/L 'CHLORIDE'
      WHEN labname = 'glucose' and le.labresult > 10000 THEN null -- mg/dL 'GLUCOSE'
      WHEN labname = 'Hct' and le.labresult >   100 THEN null -- % 'HEMATOCRIT'
      WHEN labname = 'Hgb' and le.labresult >    50 THEN null -- g/dL 'HEMOGLOBIN'
      WHEN labname = 'lactate' and le.labresult >    50 THEN null -- mmol/L 'LACTATE'
      WHEN labname = 'platelets x 1000' and le.labresult > 10000 THEN null -- K/uL 'PLATELET'
      WHEN labname = 'potassium' and le.labresult >    30 THEN null -- mEq/L 'POTASSIUM'
      WHEN labname = 'PTT' and le.labresult >   150 THEN null -- sec 'PTT'
      WHEN labname = 'PT - INR' and le.labresult >    50 THEN null -- 'INR'
      WHEN labname = 'PT' and le.labresult >   150 THEN null -- sec 'PT'
      WHEN labname = 'sodium' and le.labresult >   200 THEN null -- mEq/L == mmol/L 'SODIUM'
      WHEN labname = 'BUN' and le.labresult >   300 THEN null -- 'BUN'
      WHEN labname = 'WBC x 1000' and le.labresult >  1000 THEN null -- 'WBC'
    ELSE le.labresult
    END AS labresult
  FROM patient p
  LEFT JOIN lab le
    ON p.patientunitstayid = le.patientunitstayid
    AND le.labresultoffset <= 1440
    AND le.labname in
    (
      'anion gap',
      'albumin',
      '-bands',
      'bicarbonate',
      'HCO3',
      'total bilirubin',
      'creatinine',
      'chloride',
      'glucose',
      'Hct',
      'Hgb',
      'lactate',
      'platelets x 1000',
      'potassium',
      'PTT',
      'PT - INR',
      'PT',
      'sodium',
      'BUN',
      'WBC x 1000'
    )
    AND labresult IS NOT null AND labresult > 0 -- lab values cannot be 0 and cannot be negative
) pvt
GROUP BY pvt.patientunitstayid
ORDER BY pvt.patientunitstayid;
```

**关键改进**（相比之前的模糊匹配版本）：
1. ✅ 使用**精确匹配**（`labname = 'xxx'`）而非模糊匹配（`ILIKE`）
2. ✅ 包含**异常值检查**（`CASE WHEN labresult > xxx THEN null`）
3. ✅ 直接使用 `labname in (...)` 过滤，性能更好
4. ✅ 包含更多指标（`albumin`, `bands`, `HCO3`, `total bilirubin`, `lactate`, `sodium`）

---

## Python 模板（精确匹配 + 异常值检查）

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

# 基于 eicu-code 官方实现的 labname 映射
# 参考: D:\Project\eicu-code\concepts\labsfirstday.sql
LAB_NAME_MAP = {
    'aniongap': 'anion gap',
    'albumin': 'albumin',
    'bands': '-bands',
    'bicarbonate': 'bicarbonate',
    'hco3': 'HCO3',  # HCO3 = bicarb, but eICU has both
    'bilirubin_total': 'total bilirubin',
    'creatinine': 'creatinine',
    'chloride': 'chloride',
    'glucose': 'glucose',
    'hematocrit': 'Hct',
    'hemoglobin': 'Hgb',
    'lactate': 'lactate',
    'platelets': 'platelets x 1000',  # Note: units are x 1000
    'potassium': 'potassium',
    'ptt': 'PTT',
    'inr': 'PT - INR',  # Note: space before INR
    'pt': 'PT',
    'sodium': 'sodium',
    'bun': 'BUN',
    'wbc': 'WBC x 1000'  # Note: units are x 1000
}

# 异常值检查阈值（基于 eicu-code）
SANITY_CHECKS = {
    'albumin': lambda x: 0 <= x <= 10,  # g/dL
    'anion gap': lambda x: 0 <= x <= 10000,  # mEq/L
    '-bands': lambda x: 0 <= x <= 100,  # %
    'bicarbonate': lambda x: 0 <= x <= 10000,  # mEq/L
    'HCO3': lambda x: 0 <= x <= 10000,  # mEq/L
    'total bilirubin': lambda x: 0 <= x <= 150,  # mg/dL
    'creatinine': lambda x: 0 <= x <= 150,  # mg/dL
    'chloride': lambda x: 0 <= x <= 10000,  # mEq/L
    'glucose': lambda x: 0 <= x <= 10000,  # mg/dL
    'Hct': lambda x: 0 <= x <= 100,  # %
    'Hgb': lambda x: 0 <= x <= 50,  # g/dL
    'lactate': lambda x: 0 <= x <= 50,  # mmol/L
    'platelets x 1000': lambda x: 0 <= x <= 10000,  # K/uL
    'potassium': lambda x: 0 <= x <= 30,  # mEq/L
    'PTT': lambda x: 0 <= x <= 150,  # sec
    'PT - INR': lambda x: 0 <= x <= 50,  #
    'PT': lambda x: 0 <= x <= 150,  # sec
    'sodium': lambda x: 0 <= x <= 200,  # mEq/L
    'BUN': lambda x: 0 <= x <= 300,  #
    'WBC x 1000': lambda x: 0 <= x <= 1000,  # K/uL
}

def extract_first_day_labs(patient_ids=None, lab_names=None):
    """
    提取 ICU 第一天（0-1440 分钟）的实验室检查结果
    
    基于 eicu-code 官方实现：D:\Project\eicu-code\concepts\labsfirstday.sql
    
    Parameters:
    -----------
    patient_ids : list or None
        患者 ID 列表，None 表示所有患者
    lab_names : list or None
        要提取的实验室指标（labname），None 表示全部（使用 LAB_NAME_MAP 的值）
    
    Returns:
    --------
    df : pandas DataFrame
        长格式实验室数据
    """
    if lab_names is None:
        lab_names = list(LAB_NAME_MAP.values())
    
    # 构建 labname 过滤条件（使用精确匹配）
    lab_names_str = "','".join(lab_names)
    
    # 构建查询（基于 eicu-code）
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
    for lab_name, check_func in SANITY_CHECKS.items():
        if lab_name in lab_names:
            # 获取阈值
            if lab_name == 'albumin':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 10 THEN NULL")
            elif lab_name == 'anion gap':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 10000 THEN NULL")
            elif lab_name == '-bands':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND (l.labresult < 0 OR l.labresult > 100) THEN NULL")
            elif lab_name == 'bicarbonate' or lab_name == 'HCO3':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 10000 THEN NULL")
            elif lab_name == 'total bilirubin':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 150 THEN NULL")
            elif lab_name == 'creatinine':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 150 THEN NULL")
            elif lab_name == 'chloride':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 10000 THEN NULL")
            elif lab_name == 'glucose':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 10000 THEN NULL")
            elif lab_name == 'Hct':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 100 THEN NULL")
            elif lab_name == 'Hgb':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 50 THEN NULL")
            elif lab_name == 'lactate':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 50 THEN NULL")
            elif lab_name == 'platelets x 1000':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 10000 THEN NULL")
            elif lab_name == 'potassium':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 30 THEN NULL")
            elif lab_name == 'PTT':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 150 THEN NULL")
            elif lab_name == 'PT - INR':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 50 THEN NULL")
            elif lab_name == 'PT':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 150 THEN NULL")
            elif lab_name == 'sodium':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 200 THEN NULL")
            elif lab_name == 'BUN':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 300 THEN NULL")
            elif lab_name == 'WBC x 1000':
                case_clauses.append(f"WHEN l.labname = '{lab_name}' AND l.labresult > 1000 THEN NULL")
    
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

def pivot_labs_to_wide(df):
    """
    将长格式实验室数据转换为宽表格式（基于 eicu-code）
    
    Parameters:
    -----------
    df : pandas DataFrame
        长格式数据（来自 extract_first_day_labs）
    
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
    # 示例 1：提取所有患者的所有实验室指标
    labs = extract_first_day_labs()
    print(f"提取了 {len(labs)} 条记录")
    print(labs.head(10))
    
    # 示例 2：转换为宽表
    wide_df = pivot_labs_to_wide(labs)
    print("\n宽表格式:")
    print(wide_df.head(5))
    
    # 示例 3：只提取肌酐和 BUN
    labs_subset = extract_first_day_labs(
        lab_names=['creatinine', 'BUN']
    )
    print(f"\n提取了 {len(labs_subset)} 条肌酐和 BUN 记录")
    
    conn.close()
```

---

## 注意事项（基于 eicu-code）

### 1. 使用精确匹配（重要！）

✅ **正确** (精确匹配，基于 eicu-code):
```sql
SELECT l.patientunitstayid, l.labname, l.labresult
FROM lab l
WHERE l.labname = 'creatinine'  -- 精确匹配
  AND l.labresultoffset >= 0 AND l.labresultoffset < 1440
```

❌ **错误** (模糊匹配，不准确):
```sql
SELECT l.patientunitstayid, l.labname, l.labresult
FROM lab l
WHERE l.labname ILIKE '%creatinine%'  -- 模糊匹配，可能匹配到不需要的结果
  AND l.labresultoffset >= 0 AND l.labresultoffset < 1440
```

**原因**：eicu-code 官方实现使用精确匹配（`labname = 'xxx'`），因为 `labname` 的值是标准化的。

### 2. 异常值检查（重要！）

基于 eicu-code，查询时务必添加异常值检查：

```sql
SELECT 
    l.patientunitstayid,
    l.labname,
    l.labresult,
    CASE
      WHEN l.labname = 'creatinine' AND l.labresult > 150 THEN NULL  -- > 150 mg/dL 视为异常
      WHEN l.labname = 'albumin' AND l.labresult > 10 THEN NULL     -- > 10 g/dL 视为异常
      WHEN l.labname = 'Hct' AND l.labresult > 100 THEN NULL       -- > 100% 视为异常
      ELSE l.labresult
    END AS labresult_checked
FROM lab l
WHERE l.labresultoffset >= 0 AND l.labresultoffset < 1440
  AND l.labresult IS NOT NULL
  AND l.labname = 'creatinine'
```

**异常值阈值**（基于 eicu-code）：
- `albumin`: > 10 g/dL
- `anion gap`: > 10000 mEq/L
- `-bands`: < 0 或 > 100%
- `bicarbonate` / `HCO3`: > 10000 mEq/L
- `total bilirubin`: > 150 mg/dL
- `creatinine`: > 150 mg/dL
- `chloride`: > 10000 mEq/L
- `glucose`: > 10000 mg/dL
- `Hct`: > 100%
- `Hgb`: > 50 g/dL
- `lactate`: > 50 mmol/L
- `platelets x 1000`: > 10000 K/uL
- `potassium`: > 30 mEq/L
- `PTT`: > 150 sec
- `PT - INR`: > 50
- `PT`: > 150 sec
- `sodium`: > 200 mEq/L
- `BUN`: > 300
- `WBC x 1000`: > 1000 K/uL

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

```sql
-- 安全转换示例（基于 eicu-code）
SELECT 
    labname,
    labresult,
    CASE 
        WHEN labresult ~ '^[0-9]+\.?[0-9]*$' 
        THEN labresult::numeric 
        ELSE NULL 
    END AS labresult_numeric
FROM lab
WHERE labresult IS NOT NULL
```

### 5. labname 的标准化值（重要！）

**不要使用模糊匹配！** eICU 的 `labname` 值是标准化的，直接使用精确匹配：

| 指标 | 准确 labname | 注意事项 |
|------|-------------|------------|
| 阴离子间隙 | `'anion gap'` |  |
| 白蛋白 | `'albumin'` |  |
| bands | `'-bands'` | 注意开头的减号 |
| 碳酸氢根 | `'bicarbonate'` 或 `'HCO3'` | 两个都可能有 |
| 总胆红素 | `'total bilirubin'` |  |
| 肌酐 | `'creatinine'` |  |
| 氯 | `'chloride'` |  |
| 葡萄糖 | `'glucose'` |  |
| 红细胞压积 | `'Hct'` | 大写 H |
| 血红蛋白 | `'Hgb'` | 大写 H |
| 乳酸 | `'lactate'` |  |
| 血小板 | `'platelets x 1000'` | 注意单位是 x 1000 |
| 钾 | `'potassium'` |  |
| PTT | `'PTT'` | 大写 |
| INR | `'PT - INR'` | 注意空格和连字符 |
| PT | `'PT'` | 大写 |
| 钠 | `'sodium'` |  |
| BUN | `'BUN'` | 大写 |
| 白细胞 | `'WBC x 1000'` | 注意单位是 x 1000 |

### 6. 时间范围

- `labresultoffset`: 结果时间（相对于 ICU 入住的分钟数）
- `labresultrevisedoffset`: 修订结果时间（如有）
- 第一天 = `labresultoffset >= 0 AND labresultoffset < 1440`

### 7. 数据质量

- **缺失值多**: eICU 的实验室数据完整性低于 MIMIC-IV
- **异常值**: 查询时务必添加合理性检查（基于 eicu-code）
- **重复值**: 同一患者同一指标可能有多个记录（不同时间），使用 `MIN`, `MAX`, `AVG` 聚合

### 8. 性能优化

- `lab` 表有 **约 2500 万条记录**（非之前错误描述的 3913 万）
- 查询时务必:
  - 使用 `patientunitstayid` 过滤
  - 限定 `labresultoffset` 范围
  - 使用 `labname IN (...)` 而非 `labname ILIKE '%xxx%'`
  - 添加 `LIMIT` 子句（测试时）
  - 使用 `EXPLAIN` 分析查询计划

---

## 常用查询场景（基于 eicu-code）

### 1. 提取 AKI 相关指标（肌酐、BUN）

```sql
-- 基于 eicu-code 官方实现
SELECT 
    pvt.patientunitstayid,
    min(CASE WHEN labname = 'creatinine' THEN labresult ELSE null END) as creatinine_min,
    max(CASE WHEN labname = 'creatinine' THEN labresult ELSE null END) as creatinine_max,
    min(CASE WHEN labname = 'BUN' THEN labresult ELSE null END) as bun_min,
    max(CASE WHEN labname = 'BUN' THEN labresult ELSE null END) as bun_max
FROM
( 
  SELECT p.patientunitstayid, le.labname,
    CASE
      WHEN le.labname = 'creatinine' AND le.labresult > 150 THEN NULL
      WHEN le.labname = 'BUN' AND le.labresult > 300 THEN NULL
      ELSE le.labresult
    END AS labresult
  FROM patient p
  LEFT JOIN lab le
    ON p.patientunitstayid = le.patientunitstayid
    AND le.labresultoffset <= 1440
    AND le.labname IN ('creatinine', 'BUN')
    AND le.labresult IS NOT NULL AND le.labresult > 0
) pvt
GROUP BY pvt.patientunitstayid
ORDER BY creatinine_max DESC NULLS LAST
LIMIT 100;
```

---

## 参考链接

- **eicu-code 官方仓库**: `D:\Project\eicu-code\concepts\labsfirstday.sql`
- **lab 表结构**: https://lcp.mit.edu/eicu-schema-spy/tables/lab.html
- **eICU 数据介绍**: https://eicu-crd.mit.edu/

---

**最后更新**: 2026-05-22  
**更新人**: 悟空（基于 eicu-code 官方实现修正）
