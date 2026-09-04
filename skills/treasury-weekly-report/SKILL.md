---
name: treasury-weekly-report
display_name: 资金周报生成器
description: "生成企业集团资金管理周报——当用户提到'资金周报'、'treasury report'、'周报生成'、'资金分析'、'资金管理报告'时激活。支持从上传文件或OneDrive自动拉取数据，输出16维度分析的MD+HTML仪表盘+PDF三件套。"
icon: "💰"
trigger: 资金周报 treasury weekly report 资金管理 周报生成 资金分析
patterns:
  - {pattern: "资金周报", confidence: 0.95}
  - {pattern: "treasury.*report", confidence: 0.90}
  - {pattern: "资金管理.*报告", confidence: 0.85}
inputs:
  - name: bank_flow_file
    description: "银行流水明细Excel（含'流水明细'和'账户信息'两个sheet）"
    type: path
    required: true
  - name: investment_ledger_file
    description: "理财台账Excel（含'在投理财'和'本周到期'两个sheet）"
    type: path
    required: true
  - name: capital_plan_file
    description: "资金计划表Excel（含'本周资金计划'sheet，包含计划vs实际对比）"
    type: path
    required: true
  - name: last_week_summary_file
    description: "上周资金汇总Excel（含'上周每日汇总'和'上周关键指标'sheet）"
    type: path
    required: true
  - name: alert_rules_file
    description: "预警规则表Excel（含'账户预警规则'sheet）。首次提供后自动存入Agent Files复用"
    type: path
    required: false
    default: "agent_files/treasury/预警规则表.xlsx"
  - name: report_week
    description: "报告周次，如'W24'或'2026W24'，自动从数据中推断"
    type: string
    required: false
  - name: company_name
    description: "集团名称，用于报告标题"
    type: string
    required: false
    default: "示例集团"
tools: [run_python, run_python_with_write, open_in_session_tab, file_read, file_write, file_copy, folder_list, folder_create]
scripts: [treasury_pipeline.py]
id: 5128ee2f08354cf6b7936ee413656a48
---

## Overview

端到端资金管理周报生成技能。从5份Excel数据源（银行流水、理财台账、资金计划、预警规则、上周汇总）自动完成数据校验→16维度指标计算→风险热力图→可视化仪表盘→PDF正式版的全流程，并输出分级管理建议。适用于集团型企业（多子公司、多银行账户、多币种）的资金管理部周报场景。

## Workflow

### Step 1: 数据获取与校验
- **Mode**: `agentic`
- **Tool**: `file_read`, `run_python`
- **Input**: 用户上传的5份Excel文件路径，或OneDrive/SharePoint路径
- **Output**: 数据加载确认 + 质量报告（行数/列数/缺失值/日期范围）
- **Validate**: 5份文件全部成功读取，关键列无缺失
- **On failure**: 
  - 文件缺失：提示用户补充
  - 列名不匹配：尝试模糊匹配，失败则列出期望列名让用户确认
  - 预警规则文件未提供：从 `agent_files/treasury/预警规则表.xlsx` 读取缓存版本

**预警规则持久化逻辑：**
1. 首次运行：用户提供预警规则表 → 处理完毕后 `file_copy` 到 `agent_files/treasury/预警规则表.xlsx`
2. 后续运行：若用户未提供该文件，自动从 Agent Files 读取
3. 若用户提供新版本，覆盖旧版本

**自动推断周次：** 从银行流水的日期范围推断 `report_week`（如流水日期为6/8-6/12 → W24）。

### Step 2: 核心指标计算
- **Mode**: `deterministic`
- **Tool**: `run_python` (执行 `treasury_pipeline.py` 的 `compute_core_metrics`)
- **Input**: Step 1 加载的数据
- **Output**: 每日余额趋势、收付结构汇总、环比对比数据
- **Validate**: 日终余额合理（正数、在历史范围内），收付合计与计划表一致
- **On failure**: 若余额出现不合理值（负数/极端值），切换为仅使用计划表的宏观数据

关键计算逻辑：
- 每日余额 = 上周五收盘余额 + 累计净流量（使用计划表实际合计×流水日比例分配）
- 收付分类：基于摘要/用途字段自动映射到10+标准类别
- 环比：自动读取上周关键指标表对比

### Step 3: 异常检测
- **Mode**: `deterministic`
- **Tool**: `run_python` (执行 `detect_anomalies`)
- **Input**: 流水数据 + 预警规则表
- **Output**: 触发预警列表（W001余额低于预警线 / W003单笔超限 / W004摘要缺失 / W005异常时间）
- **Validate**: 每条预警包含完整字段（日期/子公司/金额/规则类型）
- **On failure**: 若预警规则表结构不匹配，降级为仅检测摘要缺失和大额交易

### Step 4: 风险矩阵与热力图数据
- **Mode**: `deterministic`
- **Tool**: `run_python` (执行 `compute_risk_matrix`)
- **Input**: 流水数据 + 预警规则
- **Output**: 子公司×日期百分位风险矩阵（Top15×5日） + 支付时段分布矩阵（10h×5d）
- **Validate**: 热力图数据点完整（15×5=75个 + 10×5=50个）
- **On failure**: 若子公司数不足15家，使用全部子公司

### Step 5: 扩展维度分析
- **Mode**: `agentic`
- **Tool**: `run_python`
- **Input**: 全部数据源 + Step 2-4 计算结果
- **Output**: 8个扩展维度的分析数据（应收应付账龄/外汇敞口/授信额度/资金池/滚动预测/融资利差/合规内控/套保监控）
- **Validate**: 每个维度有对应数据表或摘要
- **On failure**: 若某维度数据不足，标注"数据待补充"并跳过

扩展维度说明：
- 外汇定价使用利率平价(IRP) + Garman-Kohlhagen模型（`norm_cdf`手动实现，因sandbox无scipy）
- 账龄/授信/套保等维度需要额外数据源，首次运行时使用模拟数据并标注
- 后续可接入ERP/期货系统真实数据

### Step 6: 管理建议生成
- **Mode**: `agentic`
- **Input**: Step 2-5 全部分析结果
- **Output**: 5-8条分级管理建议（🔴紧急/🟡关注/🟢优化）
- **Validate**: 每条建议有具体数据支撑、明确责任方和时限
- **On failure**: 至少输出3条基于核心指标的建议

建议生成逻辑：
- 🔴紧急：预警事项(W001-W005)、余额逼近安全线、授信即将到期
- 🟡关注：执行偏差>15%、外汇敞口未覆盖、套保率低于目标
- 🟢优化：理财收益提升空间、闲置资金压缩、期限结构优化

### Step 7: 生成Markdown周报
- **Mode**: `deterministic`
- **Tool**: `file_write`, `open_in_session_tab`
- **Input**: 全部计算结果
- **Output**: `artifacts/{company}_W{week}_资金周报.md` — 16章完整版
- **Validate**: MD文件包含16个二级标题
- **On failure**: 检查是否有XML特殊字符（`<`/`>`/`&`）未转义

### Step 8: 生成HTML可视化仪表盘
- **Mode**: `deterministic`
- **Tool**: `file_write`, `open_in_session_tab`
- **Input**: 全部计算结果（JSON格式嵌入HTML）
- **Output**: `artifacts/{company}_W{week}_仪表盘.html` — 7标签页21+图表（含热力图）
- **Validate**: HTML文件包含Highcharts初始化代码，所有图表容器div存在
- **On failure**: 逐图表排查数据绑定

HTML技术栈：
- Highcharts 12.1.2（从 `/vendor/highcharts/` 加载，含 heatmap、solid-gauge 模块）
- 使用 `var(--color-*)` 主题变量适配深浅色模式
- Tab切换使用原生JS，无需框架

### Step 9: 生成PDF正式版
- **Mode**: `deterministic`
- **Tool**: `run_python` (ReportLab)
- **Input**: 全部计算结果 + matplotlib图表PNG
- **Output**: `artifacts/{company}_W{week}_资金周报.pdf` — 含图表的打印版
- **Validate**: PDF页数合理（8-12页），包含页眉页脚和密级标识
- **On failure**: 
  - LayoutError → 检查Paragraph文本中的`<`未转义（用`re.sub(r'<(?![/a-zA-Z])', '&lt;', text)`）
  - 图表缺失 → 确认matplotlib chart PNG已生成在artifacts/

PDF技术要点：
- 中文字体：`STSong-Light`（CID字体，无需安装）
- matplotlib图表先存为PNG（dpi=150），再用`Image()`嵌入
- 使用`KeepTogether`防止内容跨页断裂
- 页眉页脚通过`onLaterPages`回调实现

### Step 10: 沉淀本周汇总（供下期环比）
- **Mode**: `deterministic`
- **Tool**: `run_python_with_write`, `file_copy`
- **Input**: Step 2 核心指标
- **Output**: `agent_files/treasury/上周汇总_W{week}.xlsx`
- **Validate**: 文件包含'每日汇总'和'关键指标'两个sheet
- **On failure**: 保存为JSON备份

## Output

每次运行产出4个文件：

| 文件 | 格式 | 说明 |
|------|------|------|
| `{company}_W{week}_资金周报.md` | Markdown | 16章完整版文字报告 |
| `{company}_W{week}_仪表盘.html` | HTML | 交互式可视化（7标签页·21图·含热力图） |
| `{company}_W{week}_资金周报.pdf` | PDF | 正式打印版（8-12页·页眉页脚·密级标识） |
| `上周汇总_W{week}.xlsx` | Excel | 沉淀至Agent Files供下期环比 |

## Lessons Learned

### Do
- **先用计划表宏观数据校准**：流水文件可能包含大量内部循环交易（归集/调拨），直接汇总会严重偏离实际。以计划表的`【合计】`行作为ground truth，流水数据仅用于日内比例分配和异常检测
- **XML转义**：所有传入ReportLab `Paragraph()` 的文本必须转义`<`为`&lt;`。使用`re.sub(r'<(?![/a-zA-Z])', '&lt;', text)`全局处理
- **分层构建PDF**：使用`doc.build(list(story))`（传入copy），支持增量构建和live preview
- **matplotlib中文**：设置`plt.rcParams['font.sans-serif'] = ['Arial Unicode MS', 'Heiti SC', 'PingFang SC']`
- **热力图百分位法**：用相对排名（百分位）而非绝对值，避免模拟数据的极端值导致热力图饱和

### Don't
- **不要用`scipy`**：sandbox中不可用，需手动实现`norm_cdf`（Abramowitz & Stegun近似）
- **不要修补已创建的Paragraph对象**：ReportLab内部有解析缓存，修改`.text`属性不会更新渲染。遇到问题必须重建整个story列表
- **不要直接汇总流水金额作为宏观指标**：内部划转会导致收支双重计算，50,000笔流水可能汇总出1800亿（实际仅38亿）
- **不要在Highcharts heatmap中使用未加载的模块**：必须先`<script src="/vendor/highcharts/modules/heatmap.js">`

### Common Failures
| 错误 | 原因 | 修复 |
|------|------|------|
| `LayoutError: Flowable too large` | Paragraph含未转义`<` | 全局`re.sub`转义 |
| `FileNotFoundError` in run_python | 分支workspace路径不同 | 使用`WORKSPACE_DIR`变量拼接路径 |
| 热力图全部100/饱和 | 使用绝对值而非百分位 | 改用日内跨公司百分位排名 |
| matplotlib中文方块 | 字体配置缺失 | 设置`font.sans-serif`列表 |
| PDF图表不显示 | PNG路径错误（分支workspace） | 在同一`run_python`中生成PNG和构建PDF |

### When to Ask the User
- 数据文件结构发生变化（新增列/改名）时需确认映射
- 首次运行外汇/套保等扩展维度时，需确认市场参数（汇率/利率/波动率）
- 预警规则阈值需要调整时
- 管理建议中涉及具体业务判断（如是否催收某客户）时
