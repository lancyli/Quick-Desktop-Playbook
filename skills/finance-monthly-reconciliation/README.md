# 💰 月末财务对账 Skill v2.0

> Amazon Quick Desktop 可复用技能 — 从数据采集到报告推送的端到端财务对账自动化

## 功能概述

一键完成月末财务对账全流程：

- 从 **OneDrive** 拉取 ERP 数据包
- 从 **Outlook 邮箱** 提取银行电子回单（PDF 附件）
- 执行**三维度自动对账**（参考号 + 金额 + 摘要匹配）
- 生成**交互式仪表盘**（6 个 Highcharts 图表）
- 生成**对账结果 Excel**（4 个 sheet）
- 通过 **Outlook / Teams / Slack** 推送报告

## 数据流

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   📁 OneDrive                   📧 Outlook 收件箱       │
│   ├── 财务月末数据包.xlsx        ├── 【工商银行】回单通知  │
│   │   ├── ERP总账明细           ├── 【建设银行】回单通知  │
│   │   ├── 银行对账单            └── ...（含PDF附件）     │
│   │   ├── 应收账款              │                       │
│   │   ├── 应付账款              │                       │
│   │   └── 资金日报              │                       │
│   │                             │                       │
│   └─── Step 1: 下载数据          └─── Step 2: 提取回单    │
│             │                             │              │
│             └──────────┬──────────────────┘              │
│                        ↓                                 │
│              🤖 Quick Desktop                            │
│              ┌─────────────────────┐                     │
│              │ Step 3: 三维度对账   │                     │
│              │  ① 参考号精确匹配    │                     │
│              │  ② 日期+金额匹配    │                     │
│              │  ③ 摘要相似度匹配   │                     │
│              └────────┬────────────┘                     │
│                       ↓                                  │
│              ┌─────────────────────┐                     │
│              │ Step 4: 分析+可视化  │                     │
│              │ Step 5: 报告生成     │                     │
│              └────────┬────────────┘                     │
│                       ↓                                  │
│              ┌─────────────────────┐                     │
│              │ Step 6: 归档+推送    │                     │
│              └────────┬────────────┘                     │
│                       ↓                                  │
│   📁 OneDrive         📧 Outlook    💬 Teams   💬 Slack  │
│   └── 对账结果.xlsx   └── 报告邮件   └── 通知   └── 通知  │
│                                                         │
└─────────────────────────────────────────────────────────┘

```

## 前置条件

### 需要连接的服务

| 连接器 | 用途 | 必需 |
| --- | --- | --- |
| **Amazon OneDrive** | 拉取 ERP 数据 / 回写对账结果 | ✅ 必需 |
| **Amazon Outlook** | 提取银行回单邮件 / 发送报告 | ✅ 必需 |
| **Amazon Teams** | 发送对账完成通知 | 可选 |
| **Slack** | 发送对账完成通知 | 可选 |

### OneDrive 目录结构

```
OneDrive/
└── 财务月末数据_202608/
    ├── 财务月末数据包_202608.xlsx    ← ERP 导出数据（5 个 sheet）
    └── 对账结果_202608.xlsx          ← 对账输出（自动回写）

```

### Outlook 邮件格式

银行回单邮件标题应包含关键词 **"电子回单通知"**，例如：

```
【中国工商银行】电子回单通知 - 08-14 付款 ¥242,100 → 星辰云科技

```

## 使用方法

### 方式一：对话触发

在 Quick Desktop 中直接对话：

```
用户: 开始本月对账
Agent: 好的，我来执行月末对账流程...
       Step 1: 从 OneDrive 下载 ERP 数据... ✅
       Step 2: 从邮箱提取银行回单... ✅ (8 份)
       Step 3: 执行三维度自动对账... ✅ (匹配率 80%)
       ...

```

### 方式二：Schedule 自动触发

配置定时任务，每月最后一个工作日自动执行：

1. 打开 **Mission Control → Schedules**
2. 创建新 Schedule：- 名称：月末财务对账

- Agent：月末财务对账助手
- 触发：每月最后一个工作日 09:00
- Prompt："请执行本月的财务月末对账流程"

## 7 步流程说明

| 步骤 | 名称 | 模式 | 工具 |
| --- | --- | --- | --- |
| 1 | 数据采集 | agentic | OneDrive 连接器 |
| 2 | 邮件回单提取 | agentic | Outlook 连接器 + PDF 解析 |
| 3 | 数据清洗+自动对账 | deterministic | run_python (pandas + difflib) |
| 4 | 智能分析+可视化 | deterministic | run_python + Highcharts |
| 5 | 报告生成 | deterministic | run_python (xlsxwriter) |
| 6 | 归档+推送 | agentic | OneDrive + Outlook + Teams + Slack |
| 7 | 定时自动化 | agentic | Schedule |

## 配置选项

| 配置项 | 默认值 | 说明 |
| --- | --- | --- |
| `onedrive_folder` | `财务月末数据_{YYYYMM}` | OneDrive 数据文件夹名 |
| `erp_filename` | `财务月末数据包_{YYYYMM}.xlsx` | ERP 数据包文件名 |
| `email_search_keyword` | `电子回单通知` | 邮件搜索关键词 |
| `email_search_days` | `31` | 搜索最近 N 天的邮件 |
| `reconciliation_tolerance` | `0.01` | 金额匹配容差（1%） |
| `similarity_threshold` | `0.6` | 摘要匹配相似度阈值 |
| `report_recipients` | `["cfo@company.com"]` | 报告邮件接收人 |

## 输出文件

| 文件 | 格式 | 内容 |
| --- | --- | --- |
| 对账结果 Excel | .xlsx | 匹配明细 / 仅ERP / 仅银行 / 汇总统计 |
| 对账仪表盘 | .html | 6 个交互图表 + KPI + 未匹配明细表 |
| 报告邮件 | HTML | 对账概况 + 关注事项 |
| 通知消息 | Markdown | Teams/Slack 简报 |

## 文件清单

```
finance_skill/
├── skill.yaml                  — Skill 定义（7 步工作流）
├── reconciliation_engine.py    — 核心对账引擎（完整 Python 代码）
├── dashboard_template.html     — 仪表盘 HTML 模板（Highcharts）
└── README.md                   — 本文件

```

## 版本历史

| 版本 | 日期 | 变更 |
| --- | --- | --- |
| 2.0.0 | 2026-09-03 | 重构数据流：ERP→OneDrive, 回单→Outlook; 优化对账引擎 |
| 1.0.0 | 2026-09-03 | 初始版本 |

