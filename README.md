# eICU-Skill

Query the eICU Collaborative Research Database 2.0 / 查询 eICU 协作研究数据库 2.0。

Generates SQL and Python code for extracting ICU data (vital signs, labs, GCS, diagnoses) from PostgreSQL. Supports both direct SQL queries and Python (psycopg2) scripts.

## 🚀 一键安装（推荐）

### OpenClaw / QClaw 用户

```bash
# 方法1：通过 skillhub 一键安装（推荐）
skillhub install eicu-skill

# 方法2：通过 ClawHub 安装
openclaw skill install https://clawhub.ai/skills/eicu-skill

# 方法3：本地安装（已下载 .skill 文件）
openclaw skill install ./eicu-skill.skill
```

### Claude Code 用户

**方法1：从统一市场安装（推荐）**

```bash
# 步骤1：添加 OAMD 插件市场（包含 mimic-skill 和 eicu-skill）
/plugin marketplace add https://github.com/yongfanbeta/OAMD-marketplace

# 步骤2：安装 eicu-skill
/plugin install eicu-skill

# 步骤3：重载插件系统
/reload-plugins
```

**方法2：从 GitHub 仓库直接安装（单个仓库）**

```bash
# 添加单个仓库作为 marketplace
/plugin marketplace add https://github.com/yongfanbeta/eicu-skill

# 安装插件
/plugin install eicu-skill
```

**方法3：本地开发模式（测试用）**

```bash
# 克隆仓库到本地
git clone https://github.com/yongfanbeta/eicu-skill.git ~/.claude/plugins/eicu-skill

# 启动 Claude Code 并加载插件
claude --plugin-dir ~/.claude/plugins/eicu-skill
```

### Codex 用户

**Codex App：**
1. 点击侧边栏的 **Plugins**
2. 搜索 `eicu-skill`
3. 点击 **+** 安装

**Codex CLI：**
```bash
# 搜索插件
/plugins

# 搜索 eicu-skill
eicu-skill

# 选择 Install Plugin
```

### Cursor 用户

在 Cursor Agent 聊天中输入：
```bash
/add-plugin eicu-skill
```

或者：
1. 打开插件市场
2. 搜索 `eicu-skill`
3. 点击安装

### 手动安装（所有智能体通用）

1. 从 [Releases](https://github.com/yongfanbeta/eicu-skill/releases) 下载 `eicu-skill.skill`
2. 将文件重命名为 `eicu-skill.zip` 并解压
3. 将解压后的 `eicu-skill/` 文件夹复制到智能体的 skills 目录（例如 `~/.qclaw/skills/`）
4. 重启智能体

## 📖 使用方法

```python
# 示例：查询 eICU 数据
请帮我提取 eICU 数据库中第一个 ICU 停留的生命体征数据
```

## 🔍 支持的查询类型

| 查询类型 | 说明 |
|---------|------|
| 患者基本信息 | patient 表（年龄、性别、种族、APACHE IV 诊断等） |
| 生命体征（周期性） | vitalperiodic 表（心率、血压、体温等，每1-5分钟记录） |
| 生命体征（非周期性） | vitalaperiodic 表（有创血流动力学监测数据） |
| 生命体征（护理记录） | nursecharting 表（护理记录中的生命体征） |
| 体格检查（GCS等） | physicalexam 表（格拉斯哥昏迷评分等） |
| 实验室检查 | lab 表（白蛋白、肌酐、血糖、电解质等） |
| 血气分析 | 可从 lab 表或动脉血气相关字段提取 |
| 尿量输出 | urineoutput 表 |
| 血管活性药物 | vasopressor 表（去甲肾上腺素、多巴胺等） |
| 输液记录 | infusion 表 |
| 体重 | weight 表 |
| 氧合指标 | 可从 vitalperiodic 或 lab 表提取 |
| 诊断与 APACHE 分组 | patient 表的 apacheadmissiondx 字段 |

## ⚠️ 重要提示

### 1. 时间表示（与 MIMIC-IV 的重要区别！）

- **eICU**：使用**偏移量（分钟）**，所有偏移量相对于 **ICU 入院时间**
  - `offset = 0`：ICU 入院时刻
  - `offset = 60`：入院后 60 分钟（1 小时）
  - `offset = 1440`：入院后 1440 分钟（24 小时）
  
- **MIMIC-IV**：使用**绝对时间戳**（`TIMESTAMP` 类型）

### 2. 关键时间字段

| 字段名 | 说明 |
|--------|------|
| `hospitaladmitoffset` | 医院入院偏移量 |
| `hospitaldischargeoffset` | 医院出院偏移量 |
| `unitdischargeoffset` | ICU 出院偏移量（**用于计算 ICU 住院时长**）|
| `observationoffset` | 生命体征记录偏移量 |
| `nursingchartoffset` | 护理记录偏移量 |
| `labresultoffset` | 实验室检查结果偏移量 |

**计算 ICU 住院时长（小时）**：
```sql
ROUND(unitdischargeoffset/60) AS icu_los_hours
```

### 3. 数据质量与性能优化

**数据质量**：
- **缺失率高**：eICU 的数据完整性低于 MIMIC-IV
- **异常值**：查询时始终添加合理性检查（例如 `heartrate BETWEEN 25 AND 225`）
- **文本值**：部分生命体征值是文本（例如 '80s' 表示"80多岁"），需要清洗

**性能优化**：

**大表**（来自 eicu-code 统计）：
- `nursecharting`：约 1.51 亿行
- `lab`：约 3913 万行
- `vitalperiodic`：约 1.46 亿行
- `vitalaperiodic`：约 2507 万行
- `patient`：约 20 万行

**优化技巧**：
1. **按 `patientunitstayid` 过滤**（最重要！）
2. **限制时间范围**（`observationoffset BETWEEN 0 AND 1440`）
3. **添加 `LIMIT` 子句**（测试时）
4. **使用 `EXPLAIN` 分析查询计划**
5. **考虑创建物化视图**（`CREATE MATERIALIZED VIEW`）

## 📚 参考资料

- [eICU 官方 SchemaSpy 文档](https://lcp.mit.edu/eicu-schema-spy/index.html)
- [eICU 数据介绍](https://eicu-crd.mit.edu/)
- [访问申请](https://physionet.org/content/eicu-crd/2.0/)
- [eICU-code 官方仓库](https://github.com/MIT-LCP/eicu-code)
- [MIMIC-code 官方仓库](https://github.com/MIT-LCP/mimic-code)

## 📦 发布文件

- **GitHub Releases**: [v1.0.0](https://github.com/yongfanbeta/eicu-skill/releases/tag/v1.0.0)
- **ClawHub**: [eicu-skill](https://clawhub.ai/skills/eicu-skill)

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

## 📄 许可证

MIT License
