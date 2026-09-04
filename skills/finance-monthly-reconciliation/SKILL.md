---
name: finance-monthly-reconciliation
display_name: 月末财务对账
description: "自动完成月末财务对账全流程——当用户提到'月末对账'、'财务对账'、'开始对账'、'对账报告'、'reconciliation'时激活。从 OneDrive 拉取 ERP 数据，从 Outlook 提取银行回单 PDF，执行三维度对账，生成仪表盘和报告，通过 Outlook/Teams/Slack 推送。"
icon: "💰"
trigger: 月末对账 财务对账 开始对账 对账报告 reconciliation 银企对账
patterns:
  - {pattern: "月末.*对账", confidence: 0.95}
  - {pattern: "财务.*对账", confidence: 0.95}
  - {pattern: "开始.*对账", confidence: 0.90}
  - {pattern: "银企.*对账", confidence: 0.90}
  - {pattern: "对账.*报告", confidence: 0.85}
inputs:
  - name: onedrive_folder
    description: "OneDrive 中存放 ERP 数据的文件夹名，如'财务月末数据_202608'"
    type: string
    required: false
    default: "财务月末数据_{YYYYMM}"
  - name: erp_filename
    description: "ERP 数据包 Excel 文件名"
    type: string
    required: false
    default: "财务月末数据包_{YYYYMM}.xlsx"
  - name: email_keyword
    description: "Outlook 中搜索银行回单邮件的关键词"
    type: string
    required: false
    default: "电子回单通知"
tools: [run_python, run_python_with_write, open_in_session_tab, file_read, file_read_pdf, file_write, folder_list, folder_create, email_send, send_self_message, send_chat_message, conversations_add_message]
scripts: [reconciliation_engine.py]
id: 32d933f4d3764152a14b23666cdbcae9
---

## Overview

端到端月末财务对账技能。从 OneDrive 拉取 ERP 数据包（5个sheet），从 Outlook 邮箱提取银行电子回单 PDF，执行三维度自动对账（参考号精确匹配 + 日期金额模糊匹配 + 摘要相似度匹配），生成交互式 Highcharts 仪表盘和对账结果 Excel，通过 Outlook 邮件、Teams、Slack 推送报告。

## 核心原则

1. **准确性第一**：所有金额计算精确到分（小数点后两位），对账匹配必须可追溯、可审计。
2. **合规优先**：遵循《企业会计准则》和企业内控制度，所有操作留有审计轨迹。
3. **主动预警**：当匹配率低于 90%、单笔差异超过 50 万、或数据异常时，立即告警并暂停等待人工确认。
4. **保密安全**：财务数据属高度敏感信息，不得泄露给非授权方，不得将数据写入非指定目录。

## 异常处理规则

| 异常情况 | 处理方式 |
|----------|----------|
| 数据文件缺失 | 列出缺失文件，通过 Teams 通知数据责任人，暂停等待 |
| 文件格式错误 | 提示具体错误（列名不匹配、编码错误等），等待修正 |
| 匹配率 < 90% | ⚠️ 黄色预警：自动分析可能原因，列出Top差异项，请求人工确认 |
| 匹配率 < 70% | 🔴 红色告警：暂停流程，通知财务经理，可能存在系统性问题 |
| 单笔差异 > 50万 | 🔴 单独告警：标记为重大差异，需逐笔人工确认 |
| 数据日期范围错误 | 提示日期范围不在目标月份，确认是否继续 |
| 连接器断连 | 提示重新认证，给出具体的设置路径指引 |

数据流：
```
📁 OneDrive(ERP数据) ──┐
                       ├→ Quick Desktop 对账引擎 → 分析 → 报告
📧 Outlook(银行回单) ──┘         │
                                ├→ 📁 OneDrive（归档对账结果）
                                ├→ 📧 Outlook（对账报告邮件）
                                ├→ 💬 Teams（完成通知）
                                └→ 💬 Slack（完成通知）
```

## Workflow

### Step 1: 数据采集 — 从 OneDrive 拉取 ERP 数据
- **Mode**: `agentic`
- **Input**: `{{onedrive_folder}}` 和 `{{erp_filename}}`
- **Output**: 本地 Excel 文件路径
- **Validate**: 文件下载成功且包含 5 个 sheet（ERP总账、银行对账单、应收账款、应付账款、资金日报）
- **On failure**: 提示用户检查 OneDrive 连接和文件路径

使用 OneDrive 连接器搜索并下载指定文件夹中的 ERP 数据包 Excel。如果 OneDrive 不可用，提示用户直接上传文件。

### Step 2: 邮件回单提取 — 从 Outlook 抓取银行电子回单
- **Mode**: `agentic`
- **Input**: `{{email_keyword}}` 搜索关键词
- **Output**: 回单 PDF 文件路径列表
- **Validate**: 搜索到包含 PDF 附件的邮件
- **On failure**: 提示用户检查邮件关键词或手动提供回单文件

在 Outlook 收件箱中搜索包含关键词的邮件（如"电子回单通知"），下载附件 PDF，使用 `file_read_pdf` 解析提取交易明细（日期、金额、收款方、摘要）。

### Step 3: 数据清洗 + 自动对账
- **Mode**: `deterministic`
- **Tool**: `run_python`
- **Input**: Step 1 的 ERP 数据 + Step 2 的回单数据
- **Output**: 对账结果 dict（matched_df, erp_only_df, bank_only_df, stats）
- **Validate**: 匹配率 > 0，stats 包含完整统计指标
- **On failure**: 检查数据格式是否正确，列名是否匹配

执行 `reconciliation_engine.py` 中的 `load_erp_data()` 加载 5 个 sheet，然后调用 `reconcile()` 进行三维度匹配：
1. **参考号精确匹配**：ERP 凭证号 PZ-xxxx ↔ 银行参考号 BK-xxxx
2. **日期+金额模糊匹配**：同日期、金额差异 ≤1%
3. **摘要相似度匹配**：difflib.SequenceMatcher 相似度 ≥0.6

### Step 4: 智能分析 + 可视化
- **Mode**: `deterministic`
- **Tool**: `run_python` → `open_in_session_tab`
- **Input**: Step 3 的对账结果 + ERP 应收应付数据
- **Output**: HTML 仪表盘文件
- **Validate**: HTML 文件生成且包含 Highcharts 图表
- **On failure**: 检查数据格式，回退到纯文本报告

调用 `generate_dashboard_data()` 生成仪表盘 JSON 数据，结合 `dashboard_template.html` 模板生成交互式仪表盘，包含：
- KPI 卡片（匹配率、未匹配数、逾期金额、资金余额）
- 对账结果饼图
- 应收/应付账龄分布
- 资金余额走势
- 部门费用分布 + 供应商 Top10
- 未匹配交易明细表

### Step 5: 报告生成
- **Mode**: `deterministic`
- **Tool**: `run_python`
- **Input**: Step 3 的对账结果
- **Output**: 对账结果 Excel + 邮件正文 HTML + 通知文本
- **Validate**: Excel 包含 4 个 sheet（匹配成功、仅ERP、仅银行、汇总统计）
- **On failure**: 逐步生成，确保至少输出 Excel

调用 `generate_excel_report()` 生成 Excel，`generate_email_body()` 生成邮件 HTML，`generate_notification_text()` 生成通知。

### Step 6: 归档 + 推送
- **Mode**: `agentic`
- **Input**: Step 4 的仪表盘 + Step 5 的报告文件
- **Output**: 发送确认
- **Validate**: 至少邮件发送成功
- **On failure**: 逐个渠道尝试，记录失败渠道

按优先级依次：
1. **Outlook 邮件**：发送对账报告（HTML 正文 + Excel 附件）
2. **Teams 消息**：发送对账完成通知摘要
3. **Slack DM**：发送对账完成通知摘要
4. **OneDrive 归档**：回写对账结果 Excel 到源文件夹

### Step 7: 定时自动化（可选）
- **Mode**: `agentic`
- **Input**: 用户确认是否设置 Schedule
- **Output**: Schedule 配置
- **Validate**: Schedule 创建成功
- **On failure**: 提供手动配置指南

建议配置：每月最后一个工作日 09:00 自动触发，Agent 选择「月末财务对账助手」。

## Output

- 📊 **交互式仪表盘** — HTML + 6 个 Highcharts 图表
- 📋 **对账结果 Excel** — 4 个 sheet（匹配成功/仅ERP/仅银行/汇总）
- 📧 **对账报告邮件** — HTML 格式含 KPI 和异常提醒
- 💬 **Teams/Slack 通知** — 关键指标摘要

## Lessons Learned

### Do
- 先做参考号精确匹配，再做日期金额匹配，最后做摘要模糊匹配（命中率递减）
- 银行回单从 Outlook 邮件获取（模拟银行发送），ERP 数据从 OneDrive 获取
- 对账结果 Excel 和仪表盘分开输出，Excel 用于归档，仪表盘用于实时查看
- 所有模拟数据使用虚构公司名（合规要求）

### Don't
- 不要把 JSON/CSV 等技术中间文件上传到 OneDrive，只放业务 Excel
- 不要把银行回单放 OneDrive（回单来源是邮箱）
- 不要在 Skill 中包含数据构造和模拟数据生成逻辑

### Common Failures
- OneDrive UploadFile 依赖 Agent Space/Federate 认证，可能报 "not signed in"，此时改为手动上传
- 银行回单 PDF 解析可能因格式差异失败，需要回退到手动输入
- 摘要匹配阈值 0.6 可能产生误匹配，建议人工确认低相似度匹配项

### When to Ask the User
- OneDrive 文件夹路径不确定时
- 匹配率异常低（<50%）时，可能数据源有问题
- 报告推送的收件人列表
- 是否设置定时 Schedule