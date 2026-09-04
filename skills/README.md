# Skills

Amazon Quick on desktop 技能包。每个技能是一个文件夹，包含 `SKILL.md`（指令集）与脚本等参考文件。

| 技能 | 触发词 | 连接器依赖 | 产出 |
|------|--------|-----------|------|
| [`treasury-weekly-report`](treasury-weekly-report/) 💰 资金周报生成器 | 资金周报 / treasury report / 资金分析 | 无（可纯本地） | MD 周报 · HTML 仪表盘 · PDF · 汇总 Excel |
| [`finance-monthly-reconciliation`](finance-monthly-reconciliation/) 📊 月末财务对账 | 月末对账 / 银企对账 / reconciliation | OneDrive · Outlook · Teams · Slack | HTML 仪表盘 · 对账 Excel · 邮件 · IM 通知 |

## 导入方式

Quick Desktop 左侧导航 **Agents & skills** → **Skills** 标签 → **+ Create** → **Import from file**，选择对应的 `SKILL.md`。导入后出现在 **MY SKILLS** 分组下。

导入前请确认：

- 技能声明的连接器已在 **Settings → Capabilities → Connectors** 中授权
- 技能要读写的本地文件夹已在 **Settings → My computer** 中授权
- 技能声明的系统工具（如 Code Execution）已在 **Settings → Capabilities → Tools** 中启用

## 技能结构约定

```
<skill-name>/
├── SKILL.md          # 必需 —— frontmatter 元数据 + Overview + Workflow + Output + Lessons Learned
├── skill.yaml        # 可选 —— 结构化配置（连接器、默认参数、工作流步骤）
├── README.md         # 可选 —— 面向人的说明文档
└── scripts/          # 可选 —— SKILL.md frontmatter 中 scripts 字段引用的脚本
```

`SKILL.md` frontmatter 关键字段：

| 字段 | 说明 |
|------|------|
| `name` | 技能标识符，与文件夹名一致 |
| `display_name` | 界面显示名 |
| `description` | 描述 + 激活条件，Quick 靠它做自动选择 |
| `trigger` / `patterns` | 触发词与正则匹配置信度 |
| `inputs` | 输入参数（名称、类型、是否必需、默认值） |
| `tools` | 技能加载时一并载入会话的工具列表 |
| `scripts` | 引用的脚本文件 |

## 编写要点

两个技能都遵循同一个设计原则：**确定性计算下沉到 Python，判断类工作留给模型**。

- 指标计算、数据匹配、图表生成 → 标记 `Mode: deterministic`，走 `run_python` 执行脚本，结果可复现、可审计
- 数据映射确认、管理建议、异常升级判断 → 标记 `Mode: agentic`，由模型处理
- 每个 Step 都写明 `Validate`（成功标准）与 `On failure`（降级路径），让技能在数据不完整时仍能产出有价值的结果

`SKILL.md` 的 **Lessons Learned** 章节记录了实际调试中踩到的坑（沙箱无 scipy、ReportLab XML 转义、Highcharts 模块加载、内部划转导致金额重复计算等），复用或改写技能前建议先读一遍。
