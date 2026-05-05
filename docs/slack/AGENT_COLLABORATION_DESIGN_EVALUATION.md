# Agent 协作机制设计方案评估

> 基于 Crewden 现有设计哲学、ROADMAP 和 smart-agent-crew 规划，对 8 条参考路线进行适配度评估。
> 每条路线给出：Crewden 已有/规划、适配度、可采纳点、不适合原因、取舍建议。
>
> 日期：2026-05-04

---

## 一、Crewden 设计哲学总结

在评估 8 条路线之前，先提炼 Crewden 的 5 条核心设计原则，这些来自 SPEC.md、ROADMAP.md 和 v1-phase-two-roadmap.md：

### 原则 1：轻角色，不重组织

> v1.0 只定义 Agent 的角色、职责、能力、工作风格、交接偏好和约束。
> 不要求：部门表、汇报线、组织架构图、manager approval 流程。

明确来自 v1-phase-two-roadmap.md：「不做复杂的人类组织架构。部门、层级、汇报关系…不应直接复制给 Agent 系统。」

### 原则 2：目标从聊天中自然产生

> 不做独立 Boss Command Center。用户仍在当前聊天里表达目标，系统应该能识别这是一个目标型请求，并在聊天内完成澄清、brief、计划、分工。

v1.2 的标题就是"Chat-Native Goal Alignment"。这个设计决策意味着 Crewden 的主 UI 是聊天，不是看板、不是项目管理 Dashboard。

### 原则 3：结构化是为了协作，不是为了填表

> Goal / Project / Task / Review / Knowledge 等结构只在能提高协作质量时引入。每个结构化对象都必须回答：解决什么协作问题？Agent 如何使用它？用户如何从中受益？

这是对"重量级管理"的明确拒绝。不为了"完整"而引入结构。

### 原则 4：Agent 自主性来自系统机制，不是 prompt

> 不要只靠 prompt 说"你要主动"。系统需要提供：inbox、task discovery、claim / handoff / escalation、reminders / SLA、review / evidence、durable context。

这是从"靠聪明 agent"到"靠可靠系统"的转变。

### 原则 5：知识层是核心能力，但先做 adapter 不做绑定

> v1.5 应优先设计知识层和 adapter 接口，不急着绑定某个外部知识库。

内置 Markdown/Git repo adapter，统一 Knowledge Adapter Interface，后续再接外部源。

---

## 二、Crewden 现有能力与规划总览

### 已完成（v0.x）

| 能力                            | 版本     | 适配度影响                          |
| ------------------------------- | -------- | ----------------------------------- |
| Agent profile、DM、mentions     | v0.4     | 路线六（Polymorphic Actor）已有基础 |
| Delegation、wake-on-demand      | v0.4.2   | 路线四（Dispatcher）已有雏形        |
| Agent-facing CLI、Internal API  | v0.4.3   | Agent 自主性基础设施已具备          |
| Reliable delivery、daemon inbox | v0.4.4   | 路线三（Inbox）有基础               |
| Agent memory workspace          | v0.4.5   | 路线三（知识库）有基础              |
| MCP bridge                      | v0.4.6   | 工具扩展通道已建立                  |
| Task Board                      | v0.6     | 路线二（Task 治理）有基础           |
| Reminders                       | v0.7     | 路线七（Autopilot）的部分原语       |
| Channel、搜索、Thread           | v0.8-0.9 | 路线六（社交协作）的基础            |

### V1.x 规划

| 版本 | 主题                        | 对应路线                  |
| ---- | --------------------------- | ------------------------- |
| v1.0 | Lightweight Agent Roles     | 路线一（轻量版）          |
| v1.1 | Goal Brief & Work Breakdown | 路线二（轻量版）          |
| v1.2 | Chat-Native Goal Alignment  | Crewden 独有设计          |
| v1.3 | Autonomous Work Loop        | 路线一 + 路线四（轻量版） |
| v1.4 | Review & Acceptance         | 路线二的质量维度          |
| v1.5 | Knowledge & Memory Layer    | 路线三（核心匹配）        |

### Smart Agent Crew 规划

| 版本 | 主题                            | 对应路线                                         |
| ---- | ------------------------------- | ------------------------------------------------ |
| v1.6 | Reliable Task Flow              | 路线二（状态机） + 路线四（retry）               |
| v1.7 | Agent Planning & Verification   | Crewden 独有（plan-execute-verify）              |
| v1.8 | Agent Memory / Skills / Runtime | 路线五（memory→skill） + 路线八（skill capture） |
| v1.9 | Orchestration DAG               | 路线二（DAG）+ 部分路线七                        |

---

## 三、8 条路线逐条评估

### 路线一：Agent 组织架构 + 自主调度（agencycli 启发）

**agencycli 模式**：Agency → Team → Role → Agent 层级 + Context Grid + Heartbeat 自调度

**Crewden 已有/规划**：

- v1.0 Lightweight Agent Roles（role, responsibilities, capabilities, workingStyle, handoffPreference, constraints）
- v1.3 Autonomous Work Loop（inbox, claim, progress, heartbeat, escalation）
- v0.4.1 daemon 恢复和自动启动

**适配度**：🟡 中 —— 核心概念有规划但形态轻得多

**可采纳点**：

| agencycli 概念              | Crewden 对应                  | 采纳建议                                                    |
| --------------------------- | ----------------------------- | ----------------------------------------------------------- |
| Role × Project Context Grid | v1.0 role + v1.1 goal context | ✅ 可参考——角色画像 + 目标上下文形成类似 Grid 的效果        |
| Heartbeat 调度器            | v1.3 autonomous work loop     | ✅ 概念一致——但 Crewden 用 inbox/claim 驱动而非 cron wakeup |
| wakeup.md 主动发现任务      | v1.3 claimable task discovery | ✅ 可参考——"idle 时主动找活"的模式                          |
| Team 层级                   | 明确不做                      | ❌ 不采纳——Crewden 明确拒绝部门/汇报线                      |

**不适合原因**：

1. Crewden 的产品原则 1 明确拒绝"部门表、汇报线、组织架构图"。`Agency → Team → Role` 的三层层级在 Crewden 的"轻角色"哲学下太重。
2. Crewden 的 UI 模型是 chat-first，不是 workspace-first。Org Chart 管理界面与 chat-first 设计冲突。
3. Crewden 已规划 v1.0 的轻量角色（role/capability/workingStyle），不需要另外建完整 Org 体系。

**取舍**：采纳 agencycli 的角色**能力画像**思路（capability-based matching），不采纳组织层级（Agency/Team）。Heartbeat 采用 Crewden 自己的 inbox/claim 模型而非 cron wakeup。

---

### 路线二：任务治理升级 + OKR/里程碑（agencycli 启发）

**agencycli 模式**：7 态状态机 + DAG + OKR 三层目标 + Milestone 进度上卷

**Crewden 已有/规划**：

- v1.6 Reliable Task Flow：严格状态机（open → in_progress → review → done、blocked → open、\* → cancelled）+ optimistic lock + 最小 DAG
- v1.1 Goal Brief & Work Breakdown：Goal → Task 拆解，不含 OKR
- v1.9 Orchestration DAG：完整 DAG 编排

**适配度**：🟡 中 —— 状态机和 DAG 规划中，OKR 不适合

**可采纳点**：

| agencycli 概念         | Crewden 对应                                                  | 采纳建议                                                    |
| ---------------------- | ------------------------------------------------------------- | ----------------------------------------------------------- |
| 7 态状态机             | v1.6 状态机（open/in_progress/review/done/blocked/cancelled） | ✅ 基本对齐——awaiting_confirmation 等价于 Crewden 的 review |
| Task DAG（depends_on） | v1.6 blockedByTaskIds + v1.9 DAG                              | ✅ 规划一致                                                 |
| OKR 三层目标           | v1.1 Goal Brief                                               | ❌ 不适合——Crewden 的原则 3 拒绝"为了完整而引入结构"        |
| Milestone 进度上卷     | v1.1 goal tasks + v1.4 review evidence                        | ⚠️ 部分——Crewden 的 Goal Brief 更轻，不做自动进度计算       |
| Task 自动触发下游      | 未规划                                                        | ⚠️ 可考虑——但 v1.9 DAG edge 机制更通用                      |

**不适合原因**：

1. OKR 是"给人看的战略工具"，Crewden 定位是"给 Agent 执行用的结构"。Goal Brief 已经覆盖了 objective/success criteria/tasks，不需要再叠一层季度 OKR。
2. Crewden 的原则 3 明确要求"每个结构化对象都必须回答：解决什么协作问题？Agent 如何使用它？"。OKR 的季度周期、加权平均、CFR 方法论在 Agent 协作场景下解决不了具体问题。
3. 进度上卷看起来好但在 chat-first UI 中难以展示——Crewden 不需要甘特图。

**取舍**：采纳 Task 严格状态机 + DAG + Goal Brief（Crewden 已规划），不采纳 OKR 三层体系。可以在 Goal Brief 中借鉴"success criteria 的度量方式"但不引入 Objective/KeyResult 层级。

---

### 路线三：Agent 间沟通 + 知识库（agencycli + Molecule 启发）

**参考模式**：Inbox 确认闸门 + Agent Card + 分层 Knowledge（LOCAL/TEAM/GLOBAL）

**Crewden 已有/规划**：

- v0.4 DM、mentions、channel messages
- v0.4.4 daemon inbox（已有 inbox 概念）
- v1.3 unified inbox（mentions + DMs + assigned tasks + claimable tasks + reminders + reviews）
- v1.5 Knowledge & Memory Layer（KnowledgeEntry, KnowledgeKind, KnowledgeAdapter）
- v1.8 per-agent memory scope

**适配度**：🟢 高 —— Crewden 已有全面规划，分层记忆完全对齐

**可采纳点**：

| 参考概念                   | Crewden 对应                             | 采纳建议                                                       |
| -------------------------- | ---------------------------------------- | -------------------------------------------------------------- | -------------- | ------ |
| Inbox 确认闸门             | v1.3 inbox + v1.4 review/approve         | ✅ 完全对齐——Crewden 的 review request + escalate 就是确认闸门 |
| Agent Card                 | v1.0 role + capabilities                 | ✅ 概念一致——用 role + capabilities 替代 Agent Card JSON       |
| LOCAL/TEAM/GLOBAL 分层记忆 | v1.5 Knowledge + v1.8 per-agent scope    | ✅ 高度对齐——scope: agent:<id>                                 | workspace:<id> | global |
| 记忆晋升 Skill             | v1.8 skill capture from successful tasks | ✅ 对齐——approve 后生成 skill draft                            |

**不适合原因**：

- 没有明显的冲突点。Crewden 的知识层设计（v1.5）和 Molecule 的分层记忆高度一致。

**取舍**：完全采纳，无需修改 Crewden 规划。分层记忆的 scope 设计（agent/workspace/global）已经在 v1.5 中体现。可以补充"记忆晋升为 Skill"的 Curator 流程（见路线五）。

---

### 路线四：Dispatcher 驱动的任务引擎（Hermes Kanban 启发）

**Hermes 模式**：中央 Dispatcher + CAS Claim + TTL + 7 列看板 + Run 历史

**Crewden 已有/规划**：

- v1.3 Autonomous Work Loop（agent 自己 claim 任务，非中央分配）
- v1.6 严格状态机 + dependency graph
- v0.4.4 reliable delivery + daemon inbox
- Paseo Runtime 作为底层（持久 turn-based 会话，非 spawn 模型）

**适配度**：🟡 中 —— 概念可参考，但架构差异大

**可采纳点**：

| Hermes 概念           | Crewden 对应                       | 采纳建议                                                             |
| --------------------- | ---------------------------------- | -------------------------------------------------------------------- |
| Dispatcher 中央调度   | v1.3 agent 自己 claim（去中心化）  | ⚠️ 方向不同——Crewden 选 agent 自驱动而非中央分配                     |
| CAS 原子领取 + TTL    | 未明确规划                         | ⚠️ 部分——Paseo 管理 session 生命周期，TTL 不那么关键                 |
| 7 列看板（含 triage） | v0.6 Task Board + v1.6 状态机      | ✅ 可参考——triage 列"原始想法"是好的补充                             |
| Run 历史记录          | v1.6 audit log                     | ✅ 部分——audit log 记录了动作，但没有"每次执行的 structured summary" |
| build_worker_context  | v1.7 plan + v1.5 knowledge context | ✅ 可参考——上下文分层组装                                            |

**不适合原因**：

1. Crewden 的架构是 Paseo 持久会话 + agent 自驱动 claim，不是 Hermes 的"dispatcher spawn worker"。引入中央 Dispatcher 会跟 v1.3 的"agent 主动认领"冲突。
2. CAS + TTL 的安全网在 Paseo 的持久会话模型下不那么关键——Paseo 本身管理 Agent 进程，不存在"子进程突然死掉"的问题。
3. Hermes 的 7 列看板是为 spawn 模型设计的（running 列显示 PID/claim），在 Paseo 持久会话下语义不同。

**取舍**：

- 采纳 7 列看板的 **状态语义**（triage/todo/ready/running/blocked/done/archived），特别是 triage 列。
- 采纳 Run 记录的 **structured summary + metadata** 设计，结合到 Crewden 的 task completion 中。
- 不采纳中央 Dispatcher 模型——Crewden 已决定走 agent 自驱动 claim（v1.3）。

---

### 路线五：Meta-Agent 自治理（Hermes Curator 启发）

**Hermes 模式**：Curator 按 cron 运行，评分/修剪/合并 skill，输出 REPORT.md

**Crewden 已有/规划**：

- v1.8 skill capture（approved task → skill draft）
- v1.9 self-improvement loop（failed→success → improvement note）
- v1.5 knowledge hygiene（stale/conflict 标记）

**适配度**：🟡 中 —— 概念有价值，但 Crewden 走的是"增量沉淀"路线而非"独立 Curator Agent"

**可采纳点**：

| Hermes 概念             | Crewden 对应                             | 采纳建议                                           |
| ----------------------- | ---------------------------------------- | -------------------------------------------------- |
| Curator 评分/修剪 skill | v1.8 skill capture + v1.5 stale/conflict | ✅ 可参考——但用"标记"替代"自动修剪"更安全          |
| 周期性 REPORT.md        | 未规划                                   | ⚠️ 可考虑——作为 Autopilot 模板                     |
| 防御保护内置 skill      | 无内置 skill                             | ❌ 暂不需要——等 Crewden 有 skill 库后再考虑        |
| Curator 独立 Agent      | v1.9 self-improvement loop               | ✅ 部分——v1.9 的 improvement note 就是轻量 Curator |

**不适合原因**：

1. Crewden 的 v1.8 走的是"每完成一个 approved task 就生成 skill draft"，是事件驱动的增量沉淀，不是定期的批量扫描。这与 Curator 的 cron 模型不同。
2. Hermes 的 Curator 是独立 meta-agent（独立进程 + 独立调度），在 Crewden 的 chat-first UI 中显得突兀。
3. Crewden 当前没有大量 skill 库需要自动修剪——先做完 v1.8 的 skill capture 再做评估。

**取舍**：采纳 Curator 的"自改进循环"概念，但落地形式是 v1.9 的 improvement note（失败的教训 → draft memory），而非独立 meta-agent。等 Crewden skill 库丰富后再考虑引入定期审查机制。

---

### 路线六：多态 Actor 社交协作模型（Multica 启发）

**Multica 模式**：actor_type + actor_id 对等模型（member/agent/system），agent 可创建 issue/发评论/被 @/被订阅/有 inbox

**Crewden 已有/规划**：

- v0.4 Agent profile + DM + mentions（Agent 已经可以在 channel 发言）
- v0.4.2 delegation（Agent 之间委派）
- v1.3 unified inbox for agents（Agent 有 inbox）
- v1.0 role + capabilities（Agent 可被发现和匹配）

**适配度**：🟢 高 —— 核心能力已有，只需补齐社交层

**可采纳点**：

| Multica 概念                    | Crewden 对应                 | 采纳建议                                                        |
| ------------------------------- | ---------------------------- | --------------------------------------------------------------- |
| Polymorphic Actor（actor_type） | 消息已有 senderName/senderId | ✅ 核心已就绪——Message 已是 polymorphic                         |
| Agent 创建 issue/task           | crewden task add CLI         | ✅ 已有——Agent 可通过 CLI 创建 task                             |
| Agent 发评论                    | Channel message + DM         | ✅ 已有——message 就是一种 comment                               |
| @mention 自动触发 task          | delegation + mentions        | ✅ 可加强——当前 delegation 是显式的，可加 mention→task 自动映射 |
| Agent 有 inbox                  | v1.3 unified inbox           | ✅ 完全对齐                                                     |
| Reaction / Subscribe            | 未规划                       | ⚠️ 可选——Chat 场景下有用，但不是核心闭环                        |

**不适合原因**：

- 几乎没有冲突。Crewden 的 Agent 从一开始就是消息的参与者（senderName），不是后来嫁接的。

**取舍**：完全采纳。不过 Reation/Subscribe 属于"nice to have"而非核心闭环——优先做 @mention → 自动 task 触发的映射。

---

### 路线七：Autopilot 自主工作引擎（Multica 启发）

**Multica 模式**：声明式规则（cron/webhook/event）+ 两种执行模式（create_issue/run_only）+ 三种并发策略

**Crewden 已有/规划**：

- v0.7 reminders（定时提醒——部分类似 schedule 触发）
- v1.3 autonomous work loop（agent 主动 check inbox/claim）
- v1.9 workflow DAG（event 驱动的编排）

**适配度**：🟢 高 —— 与 v1.3 互补，填补"定时/事件驱动"的空白

**可采纳点**：

| Multica 概念                   | Crewden 对应                            | 采纳建议                                                    |
| ------------------------------ | --------------------------------------- | ----------------------------------------------------------- |
| Schedule 触发（cron）          | v0.7 reminders                          | ✅ 可扩展——reminder 只通知，加"创建 task"动作就是 Autopilot |
| Webhook 触发                   | 未规划                                  | ✅ 新增——外部事件（GitHub PR）→ 创建 task                   |
| Event 触发                     | v1.9 DAG edge（success/failure/always） | ✅ 部分对齐——DAG edge 就是 event 触发                       |
| create_issue vs run_only       | create vs send_message 已有             | ✅ 概念一致                                                 |
| 并发策略（skip/queue/replace） | 未明确                                  | ✅ 新增加——对定时重复触发很关键                             |

**不适合原因**：

- 没有冲突。Multica 的 Autopilot 和 Crewden 的 v1.3 autonomous loop 是互补关系：Autopilot 是**外部触发**（到了时间/webhook 来了），v1.3 是**agent 内驱**（我看看有什么活）。

**取舍**：完全采纳。Autopilot 作为 v1.3 autonomous loop 的补充——"定时规则触发任务创建"+"agent 主动发现任务"=完整的自主工作体系。内置模板（Bug Triage / PR Review Reminder / Security Scan）非常有价值。

---

### 路线八：Skill 作为上下文注入的原生机制（Multica 启发）

**Multica 模式**：Skill 是纯 Markdown 文件 → daemon 注入 → provider 原生路径

**Crewden 已有/规划**：

- v1.8 skill capture（approved task → skill draft）
- v1.5 knowledge adapter（Markdown adapter）

**适配度**：🟡 中 —— skill capture 概念对齐，但注入机制不同

**可采纳点**：

| Multica 概念                    | Crewden 对应                        | 采纳建议                                             |
| ------------------------------- | ----------------------------------- | ---------------------------------------------------- |
| Skill 是 Markdown 文件          | v1.8 skill draft 是 knowledge entry | ✅ 格式一致                                          |
| Daemon 注入到 provider 原生路径 | Paseo 的 MCP bridge 提供工具        | ❌ 机制不同——Crewden 用 Paseo 而非多 provider daemon |
| Skill 与 Agent 的 n:n 关联      | v1.0 capabilities 匹配              | ✅ 可参考——capability tag 系统                       |
| 从 URL 导入 Skill               | v1.5 knowledge adapter              | ✅ 可考虑——adapter 接入外部 skill 库                 |

**不适合原因**：

1. Multica 的 skill 注入依赖"daemon 写文件到 provider 原生路径"——但 Crewden 使用 Paseo Runtime，不直接管理 agent 的本地文件系统。注入路径不同。
2. Crewden 目前没有多 provider 的本地 daemon（Paseo 统一管理），所以不需要"每个 provider 不同路径"的注入映射。

**取舍**：采纳 Skill 作为"可复用的领域知识 Markdown 文档"的概念（v1.8 已规划），但不采纳 daemon 注入机制——Crewden 改为通过 knowledge search/read API 让 agent 在 turn 开始前检索相关知识。

---

## 四、总结：8 条路线的采纳矩阵

| 路线                         | 适配度 | 采纳程度 | 核心采纳点                                                    | 核心舍弃点                            |
| ---------------------------- | ------ | -------- | ------------------------------------------------------------- | ------------------------------------- |
| **路线一** Org + Scheduler   | 🟡 中  | 部分采纳 | 角色能力画像、inbox/claim 自驱动                              | 部门/层级/汇报线/Cron wakeup          |
| **路线二** Task + OKR        | 🟡 中  | 部分采纳 | 严格状态机、DAG、Goal Brief                                   | OKR 三层体系、Milestone 进度上卷      |
| **路线三** 沟通 + 知识库     | 🟢 高  | 完全采纳 | 分层 memory scope、Inbox 闸门、Agent Card 概念                | 无明显舍弃                            |
| **路线四** Dispatcher        | 🟡 中  | 部分采纳 | 7 列看板语义（含 triage）、Run 结构化 summary、上下文分层组装 | 中央 Dispatcher、CAS+TTL、spawn 模型  |
| **路线五** Curator           | 🟡 中  | 延后评估 | improvement note 自改进循环                                   | 独立 meta-agent、定期批量修剪         |
| **路线六** Polymorphic Actor | 🟢 高  | 完全采纳 | 多态 actor、@mention→task 触发、Agent Inbox                   | Reaction/Subscribe 暂为 nice-to-have  |
| **路线七** Autopilot         | 🟢 高  | 完全采纳 | schedule/webhook/event 触发、并发策略、内置模板               | 无冲突、与 v1.3 互补                  |
| **路线八** Skill 注入        | 🟡 中  | 调整采纳 | Skill 作为 Markdown 知识文档、skill capture                   | Daemon 注入路径（改用 knowledge API） |

### 🏆 Top 3 最适配路线

1. **路线六（Polymorphic Actor）**：Crewden 已有 80% 基础，补齐 @mention→task 和统一 Activity Feed 即可达到 Multica 级别的 Agent 一等公民体验。

2. **路线七（Autopilot）**：填补 Crewden 规划中"定时/事件驱动"的空白，与 v1.3 autonomous loop 形成完美互补——Autopilot 负责"何时创建任务"，v1.3 负责"Agent 如何发现和认领"。

3. **路线三（沟通 + 知识库）**：Crewden 的 v1.5 Knowledge Layer 和 v1.3 Unified Inbox 已经覆盖了这条路线 80% 的概念。分层 memory scope 完全对齐。

### ⚠️ 需要警惕的设计陷阱

1. **不要叠床架屋**：Crewden 的原则 3 反复强调"结构只在能提高协作质量时引入"。每加一个表、一个状态、一个 UI 组件，都要回答"Agent 如何使用它"。不要因为 agencycli 有 OKR、Hermes 有 Curator、Multica 有 Reaction 就照搬。

2. **Chat-first 是硬约束**：Crewden 的 UI 主入口是聊天（v1.2 Chat-Native Goal Alignment），不是看板、不是项目管理 Dashboard、不是 Org Chart。任何新增功能必须能在聊天流中自然呈现，而不是要求用户离开聊天去另一个页面。

3. **Paseo Runtime 是硬底层**：所有"谁执行、何时执行、怎么执行"的设计必须基于 Paseo 的持久 turn-based 会话模型。不要引入 Hermes 的 spawn 子进程模型或 Multica 的 daemon claim 模型——Paseo 已经接管了 Agent 生命周期。

4. **先可靠，再聪明**：smart-agent-crew README 的第一条设计原则。v1.6 的状态机、审计、重试必须先做，否则 v1.7 的 planning、v1.8 的 skill capture、v1.9 的 DAG 编排都会放大不确定性。

---

## 五、建议的 Crewden 实施融合方案

基于上述评估，建议 Crewden v1.x 按以下优先级融合 8 条路线中的精华：

### Phase A（P0——地基）：路线二/四的核心

```
v1.6 Reliable Task Flow ← 路线二（状态机）+ 路线四（retry/escalation）
  1. Task 严格状态机：open → in_progress → review → done + blocked + cancelled
  2. 借鉴 Hermes 7列看板的 triage 列——"原始想法先入 triage，specifier 完善后 promote"
  3. Optimistic lock + audit log
  4. Daemon 失败分类（transient retry / permanent block）
  5. 最小 dependency graph
```

**为什么先做这个**：没有可靠状态机，后续的 plan/verify/review/memory 都不可靠。这是 smart-agent-crew README 第一条设计原则。

### Phase B（P1——身份 + 知识）：路线六/三/八的核心

```
v1.0 Lightweight Roles ← 路线六（Polymorphic Actor 基础）
v1.5 Knowledge Layer ← 路线三（分层 memory scope）+ 路线八（skill capture）
  1. Agent role + capabilities（已有 v1.0 规划）
  2. @mention → task 自动触发（路线六的社交协作）
  3. Unified inbox for agents（v1.3）
  4. 分层 knowledge：agent scope / workspace scope / global scope（路线三 + Molecule）
  5. Task → skill draft capture（路线八 + v1.8）
```

### Phase C（P2——目标 + 自主）：路线七/二/一的核心

```
v1.1 Goal Brief ← 路线二（Goal 替代 OKR）
v1.3 Autonomous Loop ← 路线一（inbox/claim 替代 heartbeat）
+ Autopilot ← 路线七（补充定时/事件触发）
  1. Goal Brief 替代 OKR（路线二的轻量版）
  2. Agent 自驱动 claim（v1.3，不引入中央 Dispatcher）
  3. Autopilot schedule/webhook 触发（路线七）
  4. 内置 Autopilot 模板：Bug Triage / PR Review Reminder / Weekly Report
```

### Phase D（P3——质量 + 编排）：路线五的核心

```
v1.4 Review & Acceptance
v1.9 Orchestration DAG
+ Self-improvement loop ← 路线五（轻量 Curator）
  1. Review workflow + evidence（v1.4）
  2. DAG 编排（v1.9）
  3. improvement note：失败→成功→生成改善记录（路线五轻量版）
  4. 暂不引入独立 Curator meta-agent——先积累数据
```
