# Amazon Quick Desktop Playbook

> 讲清 Amazon Quick on desktop 的能力底座、Skills 机制与企业接入方式，并在此之上沉淀可复用的业务场景方案与技能包。

**在线站点**

| 页面 | 说明 |
|------|------|
| **[Playbook 主页](https://lancyli.github.io/Quick-Desktop-Playbook/)** | 完整 Playbook：定位与价值 → 能力模型 → 场景（含已就绪资产）→ 落地四步 → 企业接入 → 资源导航 |
| [自讲解版](https://lancyli.github.io/Quick-Desktop-Playbook/overview-guided.html) | 定位与价值部分的 10 段语音讲解形态，适合直接照着讲 |
| [技术参考手册](https://lancyli.github.io/Quick-Desktop-Playbook/playbook.html) | 10 类系统工具明细、权限档位、界面速览、Skills 机制 |

---

## 这是什么

Amazon Quick on desktop 是 Amazon Quick 的原生桌面应用，对你的本地文件、已连接服务以及工作上下文有直接且持续的访问能力。

本 Playbook 分两层：

1. **平台层** —— 桌面端本身：能碰到什么数据、有哪些工具、权限怎么管、Skills 怎么运作、企业怎么接入
2. **场景层** —— 按业务域沉淀的落地方案，每个场景交付「讲解网页 + SKILL.md 技能包 + 脚本」三件套

> Amazon Quick on desktop 目前为 **Preview**，面向 Plus 与 Enterprise 账户（Free 账户前 30 天可试用）。桌面端与 Web 端共享核心 AI 能力，但 Preview 阶段两端功能集并不完全对齐。

---

## 平台层：能力底座

下表按落地时的关注点重新组织，不是官方文档的编排 —— 原始目录见[用户指南](https://docs.aws.amazon.com/quick/latest/userguide/amazon-quick-desktop.html)。

| 模块 | 说明 |
|------|------|
| **本地文件访问** | 在 `Settings → My computer` 按文件夹授权。关键词搜索、语义搜索、知识图谱抽取三项可独立开关 |
| **连接器与 MCP** | 内置 Slack / Outlook / Teams / Gmail / Google Calendar / OneDrive 等；MCP Server 支持 Local、Remote、从配置文件 Import |
| **系统工具与权限** | 10 类内置工具，每类可设 Full Access / Read Only / Ask Each Time，并可按单个操作细化 |
| **Skills** | 按需加载的自包含指令集，内置技能可整组开关，自建技能进 MY SKILLS |
| **定时任务与 Mission Control** | 周期性执行，统一查看运行状态、待办输入、KPI 与历史产出 |
| **委派编码 Agent** | 通过 ACP 协议把开发类任务派给 Kiro 或 Claude Code |
| **知识图谱与记忆** | 自动抽取人、客户、项目、事件、行动项等实体及关系，存在本地；在 `Settings → My context` 查看与配置 |
| **响应模式** | 对话与定时任务可选 Fast / Balanced / Smart（对话另有 Auto），按复杂度取舍速度与质量 |

10 类系统工具：Web Search、File operations、Browser Automation、Image Generation、Code Execution、Engram Builder、Knowledge & Memory、Agent Management、Task Management、Chat & Notifications。

几个容易踩的点：

- **Code Execution 不支持 `pip install`** —— 沙箱内只能用预装包；同一对话内变量保留；沙箱始终可访问系统临时目录
- **Browser Automation** 默认用 Chrome 配置副本；开启 `Use my Chrome` 才直连你的实时会话
- **定时任务在本地执行** —— 电脑必须开机且应用保持运行，否则该次不触发，顺延到下一个计划时间

---

## 场景层：业务场景

每个场景交付三件套：讲解网页、`SKILL.md` 技能包、确定性计算脚本。

### 财务域 ✅ 已就绪 · 2 个实践

| 实践 | 技能包 | 讲解网页 | 连接器 |
|------|--------|----------|--------|
| 💰 资金周报生成 | [`treasury-weekly-report`](skills/treasury-weekly-report/) | [单页讲解版](treasury-weekly-report-onepager.html) · [长网页版](treasury-weekly-report.html) | 无强依赖 |
| 📊 月末财务对账 | [`finance-monthly-reconciliation`](skills/finance-monthly-reconciliation/) | [自讲解版](finance-monthly-close-guided.html) · [端到端方案](finance-monthly-close.html) | OneDrive · Outlook · Teams · Slack |

**💰 资金周报生成** —— 从多份 Excel 数据源自动完成校验 → 多维度指标计算 → 异常检测 → 风险热力图 → 管理建议 → MD/HTML/PDF 三件套 → 沉淀本期汇总供下期环比。面向多子公司、多银行账户、多币种的集团型场景。

**📊 月末财务对账** —— 从 OneDrive 拉取 ERP 数据包，从 Outlook 提取银行回单 PDF，执行三维度自动对账（参考号精确匹配 → 日期+金额模糊匹配 → 摘要相似度匹配），生成仪表盘与结果 Excel，通过邮件 / Teams / Slack 推送。

### 其他业务域 · 待建

同一套方法论可直接迁移的方向，目前尚未产出技能包：

| 业务域 | 典型场景 | 所需能力 |
|--------|----------|----------|
| 📈 销售与客户经营 | 客户健康度周报、商机跟进提醒、会前背景整合 | CRM 连接器 · 邮件/日历 · 知识图谱 |
| 🔬 研究与竞品追踪 | 竞品动态周报、行业资料归集与引用整理 | Web Search · Browser Automation · 定时任务 |
| 📋 项目管理 | 项目周报、会议纪要与行动项跟踪 | IM/邮件连接器 · Activity feed · Task Management |
| 👥 人力与组织 | 招聘进展汇总、面试反馈归集、入职材料准备 | 邮件/日历 · 文档创建 · 本地文件 |
| ⚖️ 合规与法务 | 合同要点抽取、版本条款比对、检查清单核对 | 本地文件 · PDF 解析 · Code Execution |
| 🛠️ IT 与运维 | 值班日报、告警归集、变更记录整理 | MCP Server · 编码 Agent（ACP）· 定时任务 |

---

## 导入技能

```bash
git clone https://github.com/lancyli/Quick-Desktop-Playbook.git
cd Quick-Desktop-Playbook
```

在 Quick Desktop 中：

1. 左侧导航 **Agents & skills** → **Skills** 标签
2. **+ Create** → **Import from file**
3. 选择对应的 `SKILL.md`（如 `skills/treasury-weekly-report/SKILL.md`）
4. 导入后出现在 **MY SKILLS** 分组

配齐依赖：

- 在 **Settings → Capabilities → Connectors** 授权技能声明的连接器
- 在 **Settings → My computer** 授权技能要读写的本地文件夹
- 在 **Settings → Capabilities → Tools** 确认所需系统工具已启用

调用方式有三种：对话里点名、Quick 自动识别触发词、或在 Skills 列表点 **Run**。

建议先用样例数据手动跑通、核对产出，再挂定时任务。

---

## 写技能的原则

两个已就绪技能遵循同一取舍：**确定性计算下沉到脚本，判断类工作留给模型**。

- 指标计算、数据匹配、图表生成 → 标 `Mode: deterministic`，走 `run_python` 执行脚本，结果可复现、可审计
- 字段映射确认、建议生成、异常升级判断 → 标 `Mode: agentic`，由模型处理
- 每步都写明 `Validate`（成功标准）与 `On failure`（降级路径），数据不完整时仍能产出有价值结果

`SKILL.md` 的 **Lessons Learned** 章节记录了实际调试中踩到的坑，复用或改写前建议先读一遍。

---

## 目录结构

```
.
├── index.html                              # Playbook 主页 · 定位/能力/场景/落地/企业接入/资源
├── overview-guided.html                    # 扩展 · 自讲解版（10 段语音讲解）
├── playbook.html                           # 技术参考手册 · 系统工具明细 / 权限模型 / 界面
├── treasury-weekly-report-onepager.html    # 财务域 · 资金周报 · 单页讲解版
├── treasury-weekly-report.html             # 财务域 · 资金周报 · 长网页版
├── finance-monthly-close-guided.html       # 财务域 · 月末对账 · 自讲解版
├── finance-monthly-close.html              # 财务域 · 月末对账 · 端到端方案
└── skills/
    ├── README.md
    ├── treasury-weekly-report/
    │   ├── SKILL.md
    │   └── scripts/treasury_pipeline.py
    └── finance-monthly-reconciliation/
        ├── SKILL.md
        ├── skill.yaml
        ├── README.md
        └── scripts/
            ├── reconciliation_engine.py
            └── dashboard_template.html
```

---

## 企业接入 · Quick SSO

企业版用户用公司账号登录桌面端，需管理员先把组织的 OIDC 身份提供商接入 Amazon Quick。桌面端走 OIDC + PKCE，登录时浏览器需已有 Quick Web 会话才能识别所配置的 IdP。

**完整配置手册：** https://lancyli.github.io/amazon-quick-sso/ — IAM Federation + Keycloak + AD 集成指南

配置四步法：

1. 在 IdP 中注册 public OIDC client，记录 Client ID 与各 OIDC 端点
2. 在 Quick 管理控制台新增 extension access
3. 在 Quick 控制台 **Extensions** 页创建 extension
4. 装客户端点 **Continue with SSO** 验证通过后分发给用户

协议层面支持**任何兼容 OIDC 的 IdP**。官方为 Microsoft Entra ID、Google Workspace、Okta、Ping Identity（PingFederate / PingOne）提供了分步配置文档；其余 IdP（如 Keycloak）走通用 OIDC 路径，配置思路一致 —— 可参照上面的 SSO 配置手册。

账户侧身份类型不限：IAM Identity Center、IAM Federation、Quick 原生用户均可。

强制走企业 SSO —— 用 MDM 下发 `DisableSocialLogin`：

```
# macOS 偏好域
com.aws.QuickWork.mac → DisableSocialLogin = true

# Windows 注册表
HKLM\SOFTWARE\Policies\Amazon\Quick
  DisableSocialLogin (REG_DWORD) = 1
```

> IdP 中的邮箱地址必须与 Amazon Quick 里该用户的邮箱**精确匹配**，否则映射失败。应用会自动沿用操作系统代理配置，无需单独设置。

---

## 相关资源

- [Amazon Quick SSO 配置手册](https://lancyli.github.io/amazon-quick-sso/) — 姊妹站点
- [DevSecOps Playbook](https://lancyli.github.io/devsecops-playbook/) — DevOps Agent + Security Agent + Kiro-CLI 组合方案
- [Amazon Quick on desktop 用户指南](https://docs.aws.amazon.com/quick/latest/userguide/amazon-quick-desktop.html)
- [Skills and agents](https://docs.aws.amazon.com/quick/latest/userguide/skills-and-agents-desktop.html)
- [企业部署指引](https://docs.aws.amazon.com/quick/latest/userguide/desktop-enterprise-setup.html)

---

## 说明

- 内容基于 [AWS 官方文档](https://docs.aws.amazon.com/quick/latest/userguide/amazon-quick-desktop.html) 整理
- **所有示例数据、企业名称、金额均为虚构**，不含任何真实业务数据
- 技能包中的工具名与连接器名以 Quick Desktop 实际版本为准，如有差异请在导入后于技能详情页调整
