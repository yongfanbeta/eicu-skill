# eICU 2.0 输液药物查询

## 目录

1. [数据来源](#数据来源)
2. [SQL 模板](#sql-模板)
3. [Python 模板](#python-模板)
4. [注意事项](#注意事项)
5. [常用药物名称](#常用药物名称)

---

## 数据来源

eICU 的输液药物数据来自 **`infusiondrug` 表**。

**关键字段**:
- `patientunitstayid` - ICU 住院 ID
- `infusionoffset` - 输液记录相对于 ICU 入住的时间偏移（分钟）
- `drugname` - 药物名称（字符串，需要精确匹配）
- `dosage` - 剂量
- `routeadmin` - 给药途径
- `infusionrate` - 输液速率
- `infusionunit` - 输液单位

**注意**: e-icu 的 `infusiondrug` 表包含静脉输液药物（升压药、肝素、正性肌力药等）。

---

## SQL 模板

### 1. 提取患者所有输液记录

```sql
-- 查询患者的所有输液记录
SELECT
    infusionid          -- 记录 ID (PK)
  , patientunitstayid  -- 患者 ID (FK)
  , infusionoffset     -- 输液偏移时间（分钟）
  , drugname           -- 药物名称
  , dosage             -- 剂量
  , routeadmin         -- 给药途径
  , infusionrate       -- 输液速率
  , infusionunit       -- 输液单位
  , bagvolume          -- 输液袋体积
  , bagvolumeunit      -- 输液袋体积单位
FROM infusiondrug
WHERE patientunitstayid = %(patient_id)s
ORDER BY infusionoffset;
```

**字段说明**:
- `infusionoffset`: 输液记录相对于 ICU 入住的时间偏移（分钟）
- `drugname`: 药物名称（**关键字段**，用于筛选特定药物）
- `dosage`: 剂量（数值）
- `routeadmin`: 给药途径（如 'IV'，'Intravenous' 等）
- `infusionrate`: 输液速率（数值）
- `infusionunit`: 输液单位（如 'mcg/kg/min'，'ml/hr' 等）

---

### 2. 筛选特定药物（如去甲肾上腺素）

```sql
-- 筛选特定药物（如去甲肾上腺素）
SELECT
    patientunitstayid
  , infusionoffset
  , drugname
  , dosage
  , infusionrate
  , infusionunit
FROM infusiondrug
WHERE drugname IN
      (
        'Norepinephrine'
      , 'Norepinephrine ()'
      , 'Norepinephrine MAX 32 mg Dextrose 5% 250 ml (mcg/min)'
      , 'Norepinephrine MAX 32 mg Dextrose 5% 500 ml (mcg/min)'
      , 'Norepinephrine (mcg/hr)'
      , 'Norepinephrine (mcg/kg/hr)'
      , 'Norepinephrine (mcg/kg/min)'
      , 'Norepinephrine (mcg/min)'
      , 'Norepinephrine (mg/hr)'
      , 'Norepinephrine (mg/kg/min)'
      , 'Norepinephrine (mg/min)'
      , 'Norepinephrine (ml/hr)'
      , 'Norepinephrine STD 32 mg Dextrose 5% 282 ml (mcg/min)'
      , 'Norepinephrine STD 32 mg Dextrose 5% 500 ml (mcg/min)'
      , 'Norepinephrine STD 4 mg Dextrose 5% 250 ml (mcg/min)'
      , 'Norepinephrine STD 4 mg Dextrose 5% 500 ml (mcg/min)'
      , 'Norepinephrine STD 8 mg Dextrose 5% 250 ml (mcg/min)'
      , 'Norepinephrine STD 8 mg Dextrose 5% 500 ml (mcg/min)'
      , 'Norepinephrine (units/min)'
      , 'Norepinephrine (Unknown)'
      , 'norepinephrine Volume (ml)'
      , 'norepinephrine Volume (ml) (ml/hr)'
      -- Levophed (品牌名)
      , 'Levophed (mcg/kg/min)'
      , 'levophed  (mcg/min)'
      , 'levophed (mcg/min)'
      , 'Levophed (mcg/min)'
      , 'Levophed (mg/hr)'
      , 'levophed (ml/hr)'
      , 'Levophed (ml/hr)'
      , 'NSS with LEVO (ml/hr)'
      , 'NSS w/ levo/vaso (ml/hr)'
    )
  AND patientunitstayid = %(patient_id)s
ORDER BY infusionoffset;
```

**注意**: `drugname` 的值非常多样化（包括浓度、体积、单位等信息）。必须使用 **精确匹配**（`IN` 列表），不能使用模糊匹配（`LIKE '%norepinephrine%'`），否则会匹配到错误记录。

---

### 3. 创建输液药物宽表（基于 eicu-code）

eicu-code 提供了 `pivoted-infusion.sql`，将输液药物转换为宽表格式（每行一个 `infusionoffset`，每列一种药物）。

```sql
DROP TABLE IF EXISTS pivoted_infusion CASCADE;
CREATE TABLE pivoted_infusion AS
WITH vw0 AS
(
  SELECT
      patientunitstayid
    , infusionoffset
    -- 多巴胺
    , MAX(CASE WHEN drugname IN
              (
                 'Dopamine'
               , 'Dopamine ()'
               , 'DOPamine MAX 800 mg Dextrose 5% 250 ml  Premix (mcg/kg/min)'
               , 'Dopamine (mcg/hr)'
               , 'Dopamine (mcg/kg/hr)'
               , 'dopamine (mcg/kg/min)'
               , 'Dopamine (mcg/kg/min)'
               , 'Dopamine (mcg/min)'
               , 'Dopamine (mg/hr)'
               , 'Dopamine (ml/hr)'
               , 'Dopamine (nanograms/kg/min)'
               , 'DOPamine STD 15 mg Dextrose 5% 250 ml  Premix (mcg/kg/min)'
               , 'DOPamine STD 400 mg Dextrose 5% 250 ml  Premix (mcg/kg/min)'
               , 'DOPamine STD 400 mg Dextrose 5% 500 ml  Premix (mcg/kg/min)'
               , 'Dopamine (Unknown)'
              )
              THEN 1
              ELSE NULL END
            ) AS dopamine
    -- 多巴酚丁胺 (LIKE 匹配即可，无 false positive)
    , MAX(CASE WHEN LOWER(drugname) LIKE '%dobu%' THEN 1 ELSE NULL END) AS dobutamine
    -- 去甲肾上腺素
    , MAX(CASE WHEN drugname IN
              (
                 'Norepinephrine'
               , 'Norepinephrine ()'
               , 'Norepinephrine MAX 32 mg Dextrose 5% 250 ml (mcg/min)'
               , 'Norepinephrine MAX 32 mg Dextrose 5% 500 ml (mcg/min)'
               , 'Norepinephrine (mcg/hr)'
               , 'Norepinephrine (mcg/kg/hr)'
               , 'Norepinephrine (mcg/kg/min)'
               , 'Norepinephrine (mcg/min)'
               , 'Norepinephrine (mg/hr)'
               , 'Norepinephrine (mg/kg/min)'
               , 'Norepinephrine (mg/min)'
               , 'Norepinephrine (ml/hr)'
               , 'Norepinephrine STD 32 mg Dextrose 5% 282 ml (mcg/min)'
               , 'Norepinephrine STD 32 mg Dextrose 5% 500 ml (mcg/min)'
               , 'Norepinephrine STD 4 mg Dextrose 5% 250 ml (mcg/min)'
               , 'Norepinephrine STD 4 mg Dextrose 5% 500 ml (mcg/min)'
               , 'Norepinephrine STD 8 mg Dextrose 5% 250 ml (mcg/min)'
               , 'Norepinephrine STD 8 mg Dextrose 5% 500 ml (mcg/min)'
               , 'Norepinephrine (units/min)'
               , 'Norepinephrine (Unknown)'
               , 'norepinephrine Volume (ml)'
               , 'norepinephrine Volume (ml) (ml/hr)'
               -- Levophed (品牌名)
               , 'Levophed (mcg/kg/min)'
               , 'levophed  (mcg/min)'
               , 'levophed (mcg/min)'
               , 'Levophed (mcg/min)'
               , 'Levophed (mg/hr)'
               , 'levophed (ml/hr)'
               , 'Levophed (ml/hr)'
               , 'NSS with LEVO (ml/hr)'
               , 'NSS w/ levo/vaso (ml/hr)'
              )
          THEN 1 ELSE 0 END) AS norepinephrine
    -- 苯肾上腺素
    , MAX(CASE WHEN drugname IN
            (
               'Phenylephrine'
             , 'Phenylephrine ()'
             , 'Phenylephrine  MAX 100 mg Sodium Chloride 0.9% 250 ml (mcg/min)'
             , 'Phenylephrine (mcg/hr)'
             , 'Phenylephrine (mcg/kg/min)'
             , 'Phenylephrine (mcg/kg/min) (mcg/kg/min)'
             , 'Phenylephrine (mcg/min)'
             , 'Phenylephrine (mcg/min) (mcg/min)'
             , 'Phenylephrine (mg/hr)'
             , 'Phenylephrine (mg/kg/min)'
             , 'Phenylephrine (ml/hr)'
             , 'Phenylephrine  STD 20 mg Sodium Chloride 0.9% 250 ml (mcg/min)'
             , 'Phenylephrine  STD 20 mg Sodium Chloride 0.9% 500 ml (mcg/min)'
             , 'Volume (ml) Phenylephrine'
             , 'Volume (ml) Phenylephrine ()'
             -- Neo-Synephrine (品牌名)
             , 'neo-synephrine (mcg/min)'
             , 'neosynephrine (mcg/min)'
             , 'Neosynephrine (mcg/min)'
             , 'Neo Synephrine (mcg/min)'
             , 'Neo-Synephrine (mcg/min)'
             , 'NeoSynephrine (mcg/min)'
             , 'NEO-SYNEPHRINE (mcg/min)'
             , 'Neosynephrine (ml/hr)'
             , 'neosynsprine'
             , 'neosynsprine (mcg/kg/hr)'
            )
          THEN 1 ELSE 0 END) AS phenylephrine
    -- 肾上腺素
    , MAX(CASE WHEN drugname IN
            (
               'EPI (mcg/min)'
             , 'Epinepherine (mcg/min)'
             , 'Epinephrine'
             , 'Epinephrine ()'
             , 'EPINEPHrine(Adrenalin)MAX 30 mg Sodium Chloride 0.9% 250 ml (mcg/min)'
             , 'EPINEPHrine(Adrenalin)STD 4 mg Sodium Chloride 0.9% 250 ml (mcg/min)'
             , 'EPINEPHrine(Adrenalin)STD 4 mg Sodium Chloride 0.9% 500 ml (mcg/min)'
             , 'EPINEPHrine(Adrenalin)STD 7 mg Sodium Chloride 0.9% 250 ml (mcg/min)'
             , 'Epinephrine (mcg/hr)'
             , 'Epinephrine (mcg/kg/min)'
             , 'Epinephrine (mcg/min)'
             , 'Epinephrine (mg/hr)'
             , 'Epinephrine (mg/kg/min)'
             , 'Epinephrine (ml/hr)'
            )
          THEN 1 ELSE 0 END) AS epinephrine
    -- 血管加压素
    , MAX(CASE WHEN drugname IN
            (
               'Vasopressin'
             , 'Vasopressin ()'
             , 'Vasopressin 20 Units Sodium Chloride 0.9% 100 ml (units/hr)'
             , 'Vasopressin 20 Units Sodium Chloride 0.9% 250 ml (units/hr)'
             , 'Vasopressin 40 Units Sodium Chloride 0.9% 100 ml (units/hr)'
             , 'Vasopressin 40 Units Sodium Chloride 0.9% 100 ml (units/kg/hr)'
             , 'Vasopressin 40 Units Sodium Chloride 0.9% 100 ml (units/min)'
             , 'Vasopressin 40 Units Sodium Chloride 0.9% 100 ml (Unknown)'
             , 'Vasopressin 40 Units Sodium Chloride 0.9% 200 ml (units/min)'
             , 'Vasopressin (mcg/kg/min)'
             , 'Vasopressin (mcg/min)'
             , 'Vasopressin (mg/hr)'
             , 'Vasopressin (mg/min)'
             , 'vasopressin (ml/hr)'
             , 'Vasopressin (ml/hr)'
             , 'Vasopressin (units/hr)'
             , 'Vasopressin (units/kg/min)'
             , 'vasopressin (units/min)'
             , 'Vasopressin (units/min)'
             , 'VAsopressin (units/min)'
             , 'Vasopressin (Unknown)'
            )
          THEN 1 ELSE 0 END) AS vasopressin
    -- 米力农 (正性肌力药)
    , MAX(CASE WHEN drugname IN
            (
               'Milrinone'
             , 'Milrinone ()'
             , 'Milrinone (mcg/kg/hr)'
             , 'Milrinone (mcg/kg/min)'
             , 'Milrinone (ml/hr)'
             , 'Milrinone (Primacor) 40 mg Dextrose 5% 200 ml (mcg/kg/min)'
             , 'Milronone (mcg/kg/min)'
             , 'primacore (mcg/kg/min)'
            )
          THEN 1 ELSE 0 END) AS milrinone
    -- 肝素
    , MAX(CASE WHEN drugname IN
            (
               'Hepain (ml/hr)'
             , 'Heparin'
             , 'Heparin ()'
             , 'Heparin 25,000 Unit/D5w 250 ml (ml/hr)'
             , 'Heparin 25000 Units Dextrose 5% 500 ml  Premix (units/hr)'
             , 'Heparin 25000 Units Dextrose 5% 500 ml  Premix (units/kg/hr)'
             , 'Heparin 25000 Units Dextrose 5% 950 ml  Premix (units/kg/hr)'
             , 'HEPARIN #2 (units/hr)'
             , 'Heparin 8000u/1L NS (ml/hr)'
             , 'Heparin-EKOS (units/hr)'
             , 'Heparin/Femoral Sheath   (units/hr)'
             , 'Heparin (mcg/kg/hr)'
             , 'Heparin (mcg/kg/min)'
             , 'Heparin (ml/hr)'
             , 'heparin (units/hr)'
             , 'Heparin (units/hr)'
             , 'HEPARIN (units/hr)'
             , 'Heparin (units/kg/hr)'
             , 'Heparin (Unknown)'
             , 'Heparin via sheath (units/hr)'
             , 'Left  Heparin (units/hr)'
             , 'NSS carrier heparin (ml/hr)'
             , 'S-Heparin (units/hr)'
             , 'Volume (ml) Heparin-heparin 25,000 units in 0.45 % sodium chloride 500 mL infusion'
             , 'Volume (ml) Heparin-heparin 25,000 units in 0.45 % sodium chloride 500 mL infusion (ml/hr)'
             , 'Volume (ml) Heparin-heparin 25,000 units in dextrose 500 mL infusion'
             , 'Volume (ml) Heparin-heparin 25,000 units in dextrose 500 mL infusion (ml/hr)'
             , 'Volume (ml) Heparin-heparin infusion 2 units/mL in 0.9% sodium chloride (ARTERIAL LINE)'
             , 'Volume (ml) Heparin-heparin infusion 2 units/mL in 0.9% sodium chloride (ARTERIAL LINE) (ml/hr)'
            )
          THEN 1 ELSE 0 END) AS heparin
  FROM infusiondrug
  GROUP BY patientunitstayid, infusionoffset
)
SELECT
    patientunitstayid
  , infusionoffset AS chartoffset
  , dopamine::SMALLINT AS dopamine
  , dobutamine::SMALLINT AS dobutamine
  , norepinephrine::SMALLINT AS norepinephrine
  , phenylephrine::SMALLINT AS phenylephrine
  , epinephrine::SMALLINT AS epinephrine
  , vasopressin::SMALLINT AS vasopressin
  , milrinone::SMALLINT AS milrinone
  , heparin::SMALLINT AS heparin
FROM vw0
-- 至少一种药物为 1
WHERE dopamine = 1
   OR dobutamine = 1
   OR norepinephrine = 1
   OR phenylephrine = 1
   OR epinephrine = 1
   OR vasopressin = 1
   OR milrinone = 1
   OR heparin = 1
ORDER BY patientunitstayid, infusionoffset;
```

**输出表结构** (`pivoted_infusion`):
- `patientunitstayid` - ICU 住院 ID
- `chartoffset` - 时间偏移（分钟）
- `dopamine` - 是否使用多巴胺 (0/1)
- `dobutamine` - 是否使用多巴酚丁胺 (0/1)
- `norepinephrine` - 是否使用去甲肾上腺素 (0/1)
- `phenylephrine` - 是否使用苯肾上腺素 (0/1)
- `epinephrine` - 是否使用肾上腺素 (0/1)
- `vasopressin` - 是否使用血管加压素 (0/1)
- `milrinone` - 是否使用米力农 (0/1)
- `heparin` - 是否使用肝素 (0/1)

---

## Python 模板

```python
import psycopg2
import pandas as pd

# 数据库连接参数
conn_params = {
    'host': 'localhost',
    'port': 5432,
    'database': 'eicu',
    'user': 'your_username',
    'password': 'your_password'
}

def get_infusion_drugs(patient_id, drug_name=None):
    """
    获取患者的输液药物记录
    
    Parameters:
    -----------
    patient_id : int
        ICU 住院 ID (patientunitstayid)
    drug_name : str, optional
        药物名称（用于筛选特定药物）
        
    Returns:
    --------
    df : pandas DataFrame
        输液药物记录
    """
    if drug_name:
        query = """
            SELECT 
                infusionoffset,
                drugname,
                dosage,
                routeadmin,
                infusionrate,
                infusionunit
            FROM infusiondrug
            WHERE patientunitstayid = %(pid)s
              AND drugname = %(drug)s
            ORDER BY infusionoffset
        """
        params = {'pid': patient_id, 'drug': drug_name}
    else:
        query = """
            SELECT 
                infusionoffset,
                drugname,
                dosage,
                routeadmin,
                infusionrate,
                infusionunit
            FROM infusiondrug
            WHERE patientunitstayid = %(pid)s
            ORDER BY infusionoffset
        """
        params = {'pid': patient_id}
    
    with psycopg2.connect(**conn_params) as conn:
        df = pd.read_sql_query(query, conn, params=params)
    
    return df

def get_vasopressors(patient_id):
    """
    获取患者的升压药记录（常见升压药）
    
    Parameters:
    -----------
    patient_id : int
        ICU 住院 ID (patientunitstayid)
        
    Returns:
    --------
    df : pandas DataFrame
        升压药记录
    """
    # 常见升压药的 drugname 列表（部分示例）
    vasopressor_names = [
        'Norepinephrine', 'Norepinephrine ()',
        'Dopamine', 'Dopamine ()',
        'Epinephrine', 'Epinephrine ()',
        'Phenylephrine', 'Phenylephrine ()',
        'Vasopressin', 'Vasopressin ()'
    ]
    
    query = """
        SELECT 
            infusionoffset,
            drugname,
            dosage,
            infusionrate,
            infusionunit
        FROM infusiondrug
        WHERE patientunitstayid = %(pid)s
          AND drugname IN %(drugs)s
        ORDER BY infusionoffset
    """
    
    with psycopg2.connect(**conn_params) as conn:
        df = pd.read_sql_query(query, conn, params={'pid': patient_id, 'drugs': tuple(vasopressor_names)})
    
    return df

# 使用示例
if __name__ == '__main__':
    patient_id = 123456
    
    # 获取所有输液记录
    all_infusions = get_infusion_drugs(patient_id)
    print(f"Got {len(all_infusions)} infusion records")
    print(all_infusions.head())
    
    # 获取升压药记录
    vasopressors = get_vasopressors(patient_id)
    print(f"\nGot {len(vasopressors)} vasopressor records")
    print(vasopressors.head())
```

---

## 注意事项

### 1. `drugname` 精确匹配

**必须使用精确匹配**（`drugname IN (...)`），不能使用模糊匹配（`LIKE '%norepinephrine%'`）:

- ✅ **正确**: `drugname IN ('Norepinephrine', 'Levophed (mcg/min)', ...)`
- ❌ **错误**: `drugname LIKE '%norepinephrine%'`

原因：`drugname` 包含浓度、体积、单位等信息（如 `'Norepinephrine MAX 32 mg Dextrose 5% 250 ml (mcg/min)'`），模糊匹配会匹配到无关记录。

### 2. 药物剂量单位

`infusionunit` 字段包含多种单位：
- `mcg/kg/min` - 微克/公斤/分钟（常用升压药单位）
- `mcg/min` - 微克/分钟
- `mg/hr` - 毫克/小时
- `ml/hr` - 毫升/小时
- `units/hr` - 单位/小时（如肝素）
- `units/min` - 单位/分钟

**分析剂量时需注意单位转换**。

### 3. 数据缺失

- `dosage`、`infusionrate` 字段可能有 NULL 值
- 部分记录只有药物名称，没有剂量信息

### 4. 时间偏移

- `infusionoffset` 是相对于 **ICU 入住时间** 的分钟数
- `infusionoffset = 0` 表示 ICU 入住时刻
- `infusionoffset = 60` 表示入住后 60 分钟（1 小时）

---

## 常用药物名称

### 升压药 (Vasopressors)

| 通用名 | 品牌名 | 常见 `drugname` 值 |
|--------|--------|---------------------|
| **Norepinephrine** (去甲肾上腺素) | Levophed | `'Norepinephrine'`, `'Levophed (mcg/min)'` |
| **Dopamine** (多巴胺) | - | `'Dopamine'`, `'Dopamine (mcg/kg/min)'` |
| **Epinephrine** (肾上腺素) | Adrenalin | `'Epinephrine'`, `'EPI (mcg/min)'` |
| **Phenylephrine** (苯肾上腺素) | Neo-Synephrine | `'Phenylephrine'`, `'neo-synephrine (mcg/min)'` |
| **Vasopressin** (血管加压素) | - | `'Vasopressin'`, `'Vasopressin (units/hr)'` |
| **Dobutamine** (多巴酚丁胺) | Dobutrex | `'Dobutamine'`, `'Dobutrex'` |
| **Milrinone** (米力农) | Primacor | `'Milrinone'`, `'Primacor'` |

### 抗凝药

| 通用名 | 常见 `drugname` 值 |
|--------|---------------------|
| **Heparin** (肝素) | `'Heparin'`, `'Heparin (units/hr)'` |

### 其他静脉药物

- **镇静药**: 可能在 `infusiondrug` 表中，也可能在 `treatment` 表中（需检查）
- **止痛药**: 可能在 `infusiondrug` 表中，也可能在 `treatment` 表中（需检查）

---

## 参考链接

- **eicu-code 官方仓库**: [pivoted-infusion.sql](https://github.com/MIT-LCP/eicu-code/blob/master/concepts/pivoted/pivoted-infusion.sql)
- **eICU SchemaSpy**: https://lcp.mit.edu/eicu-schema-spy/index.html

---

**最后更新**: 2026-05-22  
**更新人**: 悟空（基于 MIT-LCP/eicu-code 仓库）
