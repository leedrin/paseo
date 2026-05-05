# Agent 协作机制设计方案合集

> 基于 agencycli（chenhg5/agencycli）、Hermes Kanban（NousResearch/hermes-agent）、Multica（multica-ai/multica）三个项目的深度调研，从三种不同设计哲学出发，提出 8 条互补的 Agent 协作机制设计路线。
>
> 日期：2026-05-04
> 范围：Paseo × Crewden 整合后的 Agent 协作层扩展

---

## 目录

1. [三项目核心设计哲学对比](#一三项目核心设计哲学对比)
2. [路线一：Agent 组织架构 + 自主调度（agencycli 启发）](#二路线一agent-组织架构--自主调度agencycli-启发)
3. [路线二：任务治理升级 + OKR/里程碑（agencycli 启发）](#三路线二任务治理升级--okr里程碑agencycli-启发)
4. [路线三：Agent 间沟通 + 知识库（agencycli + Molecule 启发）](#四路线三agent-间沟通--知识库agencycli--molecule-启发)
5. [路线四：Dispatcher 驱动的任务引擎（Hermes Kanban 启发）](#五路线四dispatcher-驱动的任务引擎hermes-kanban-启发)
6. [路线五：Meta-Agent 自治理（Hermes Curator 启发）](#六路线五meta-agent-自治理hermes-curator-启发)
7. [路线六：多态 Actor 社交协作模型（Multica 启发）](#七路线六多态-actor-社交协作模型multica-启发)
8. [路线七：Autopilot 自主工作引擎（Multica 启发）](#八路线七autopilot-自主工作引擎multica-启发)
9. [路线八：Skill 作为上下文注入的原生机制（Multica 启发）](#九路线八skill-作为上下文注入的原生机制multica-启发)
10. [八条路线融合总图](#十八条路线融合总图)
11. [实施优先级建议](#十一实施优先级建议)

---

## 一、三项目核心设计哲学对比

| 维度                 | agencycli                             | Hermes Kanban                    | Multica                                 |
| -------------------- | ------------------------------------- | -------------------------------- | --------------------------------------- |
| **核心隐喻**         | 公司 HR 管理                          | 工厂任务调度                     | 社交协作平台                            |
| **Agent 地位**       | 员工（有 Role/Team/Heartbeat）        | 任务执行者（被 Dispatcher 分配） | 一等公民（和人完全对等）                |
| **谁驱动工作**       | Agent heartbeat 自驱动                | 中央 Dispatcher 按 tick 分配     | 人分配 + Autopilot 定时 + @mention 触发 |
| **Agent 能创建什么** | 任务                                  | 不可创建任务                     | 可以创建 issue、发评论、加反应、订阅    |
| **管理深度**         | 深（OKR/Milestone/Role/Skill）        | 中（7 列看板 + Run 历史）        | 轻（Issue/Project/Label——像 Linear）    |
| **身份模型**         | 固定 Agent 身份（name + role + team） | Profile 绑定                     | 多态 Actor（member/agent/system）       |
| **用户规模**         | 单人/小团队                           | 单人/多 Profile                  | 多用户团队协作                          |
| **上下文传递**       | Role × Project 分层注入               | build_worker_context 分层组装    | Skill 文件注入 + 会话恢复               |
| **协作模型**         | Inbox 异步消息 + 确认闸门             | 通知订阅（IM 推送）              | @mention + Subscribe + Reaction + Inbox |
| **安全机制**         | 文件锁 + 状态机                       | CAS + TTL + 心跳续期 + PID 检测  | 工作目录隔离 + 环境变量过滤             |
| **自改进能力**       | 路线图中规划                          | Curator Meta-Agent 已落地        | 无（Skill 静态）                        |
| **部署模式**         | 单机 CLI                              | 本地 Gateway                     | Cloud-first + 本地 Daemon               |

---

## 二、路线一：Agent 组织架构 + 自主调度（agencycli 启发）

**核心思想**：引入 `Agency → Team → Role → Agent` 的组织结构，让每个 Agent 通过 Role 获得上下文，通过 Heartbeat 实现自主工作。

### 2.1 新增概念

```
Agency（公司）
  └── Team（部门）
       └── Role（岗位）
            └── Agent（员工）
```

每个 Agent 通过两层上下文获得"我是谁，我要怎么做"：

- **Role 上下文**：岗位职责、技能、行为准则（水平维度）
- **Project 上下文**：项目背景、代码库结构、技术栈（垂直维度）

### 2.2 Org Chart 数据模型

```typescript
interface Agency {
  id: string;
  name: string;
  vision: string; // 使命愿景，注入所有 Agent
}

interface Team {
  id: string;
  name: string;
  parentTeamId?: string; // 支持嵌套
}

interface Role {
  id: string;
  name: string; // e.g., "senior-frontend-dev"
  teamId: string;
  description: string; // system prompt 附加
  skills: string[]; // 关联的 skill id
  defaultModel: string; // 推荐模型
  heartbeatInterval: string; // e.g., "30m"
}
```

### 2.3 Heartbeat 调度器

Agent 按 cron 自主唤醒的模式：

```
Agent 的 Wakeup 周期:
  1. 唤醒 → 检查 inbox（未读消息 + 确认请求）
  2. 排空任务队列（按优先级 + 依赖）
  3. 队列空时 → 执行 wakeup.md（主动找活：扫描 issues、检查 PR、查看里程碑）
  4. 进入 idle，等待下次心跳
```

### 2.4 Wakeup.md 模板

```markdown
# wakeup.md (Role: qa-engineer, heartbeat: 30min)

## 每次唤醒时执行：

1. 检查 inbox 中的确认请求（优先处理）
2. 查看自己的 pending 任务队列
3. 如果队列为空：
   a. 扫描 GitHub: 新开的 PR → 创建 review 任务
   b. 扫描 GitHub: 新开的 bug issue → 创建 triage 任务给 PM
   c. 查看关联里程碑进度 → 如有阻塞，通知 PM
```

### 2.5 Context Grid（agencycli 六支柱之一）

上下文从两个轴组合——角色（horizontal）和项目（vertical）。每个 Agent 在 `hire` 时自动合并 `agency → team → role → project` 上下文。

```
     ┌─────────────────────────────────────────┐
     │         Project A    Project B    ...    │
     │  Role A  [Context A1]  [Context A2]      │
     │  Role B  [Context B1]  [Context B2]      │
     │  ...                                     │
     └─────────────────────────────────────────┘
```

### 2.6 Crewden 改造影响

| 改动                                  | 范围                | 工作量 |
| ------------------------------------- | ------------------- | ------ |
| 新增 Agency/Team/Role 表              | Crewden Store       | 1 周   |
| Role 上下文注入到 Agent system prompt | PaseoRuntimeService | 3 天   |
| Heartbeat 调度器                      | 新服务              | 1 周   |
| wakeup.md 模板系统                    | 新功能              | 3 天   |
| Web UI 组织管理页面                   | Crewden Web         | 1 周   |

---

## 三、路线二：任务治理升级 + OKR/里程碑（agencycli 启发）

**核心思想**：升级 Task 模型，从 3 态简单流转变为 7 态完整生命周期，增加依赖图、里程碑锚定、OKR 目标对齐。

### 3.1 Task 状态机升级

```
当前（3态）：todo → in_progress → done

升级后（7态）：
pending → in_progress → done_success
                      → done_failed → (retry) → pending
                      → awaiting_confirmation → done_success (人工回复)
                      → blocked → (依赖就绪) → in_progress
        → cancelled
```

新增的 `awaiting_confirmation` 是关键——Agent 遇到不确定决策时，暂停任务、归档、推送到人工 inbox，人工通过 inbox reply 决策后 Agent 在下一次唤醒时继续。

### 3.2 Task 依赖图（DAG）

```typescript
interface Task {
  id: string;
  // ... 现有字段
  dependsOn: string[]; // 前置 task id 列表

  // 完成后自动触发
  onSuccess: {
    createTasks: TaskTemplate[]; // 自动创建下游任务
    notifyAgents: string[]; // 通知相关 Agent
  };

  // 关联目标
  milestoneId?: string; // 归属里程碑
  okrKeyResultId?: string; // 直接关联的 OKR KR
}
```

### 3.3 OKR + Milestone 三层目标模型

```
Vision（公司愿景，注入所有 Agent context）
  └── OKR（季度目标，3-5 个 Objective × 2-4 Key Result）
       └── Milestone（项目里程碑，含 DoD 完成标准）
            └── Task（具体任务，关联到 Milestone）
```

进度自动上卷：

```
Task done → Milestone.progress = completedTasks / totalTasks
          → KR.currentValue 更新
          → Objective.progress = weightedAvg(KRs)
```

Agent 在上下文中感知目标：

```markdown
<!-- 自动注入到 Agent 的 AGENTS.md -->

## 🎯 当前目标

### Q2 OKR: 建立可靠的多 Agent 自主运营体系

- KR1: 任务成功率 90% (当前: 72%) ⚠️
- KR2: 每周 50+ 自动任务 (当前: 33) ⚠️

### 项目里程碑

- [65%] v1.0 稳定版发布 (Due: May 1)
- [40%] 全平台 IM 接入 (Due: Apr 20) 🔴 即将到期

> 优先处理与上述目标相关的任务。
```

### 3.4 Crewden 改造影响

| 改动                          | 范围                      | 工作量 |
| ----------------------------- | ------------------------- | ------ |
| Task 状态机扩展（5 个新状态） | Crewden Store + Task 服务 | 1 周   |
| Task DAG 依赖图               | Crewden Store + Task 服务 | 3 天   |
| OKR 模型 + 进度上卷           | 新增模块                  | 1 周   |
| Milestone 模型 + Task 关联    | 新增模块                  | 3 天   |
| Agent 上下文中注入目标        | PaseoRuntimeService       | 2 天   |
| Web UI 看板多列 + 里程碑视图  | Crewden Web               | 2 周   |

---

## 四、路线三：Agent 间沟通 + 知识库（agencycli + Molecule 启发）

**核心思想**：引入异步 inbox 确认闸门 + Agent Card 能力声明 + 分层知识库。

### 4.1 三层沟通模型

| 层级                       | 当前 Crewden | 需新增    | 用途                     |
| -------------------------- | ------------ | --------- | ------------------------ |
| **Channel**（广播聊天）    | ✅ 已有      | —         | 团队公开讨论、进展同步   |
| **DM**（点对点消息）       | ✅ 已有      | —         | Agent 间私下协调         |
| **Inbox**（异步任务确认）  | ❌ 缺失      | ✅ 需新增 | Agent → 人工暂停请求决策 |
| **Agent Card**（能力声明） | ❌ 缺失      | ✅ 需新增 | Agent 间相互发现和委托   |

### 4.2 Inbox 确认闸门

```
Agent 执行任务中遇到不确定决策
  → task confirm-request --summary "PR #42 发现潜在 SQL 注入，是否放行？"
  → 任务状态变为 awaiting_confirmation，归档
  → 消息进入人工 inbox（Web UI 显式标记）

人工在 Web UI 查看 inbox → 回复 "需要修复" 或 "可以放行"
  → Agent 下次唤醒时读取 inbox reply
  → 继续执行或标记失败
```

### 4.3 Agent Card（能力声明）

每个 Agent 自动生成：

```json
{
  "name": "dev-claude",
  "role": "senior-backend-dev",
  "team": "engineering",
  "skills": ["go", "postgres", "grpc", "k8s"],
  "preferredTasks": ["feature", "bug"],
  "currentLoad": 3,
  "status": "working",
  "lastActiveAt": "2026-05-04T10:30:00Z"
}
```

其他 Agent 可以通过 `crewden_list_agents` 查看所有 Agent Card，在委托任务时选择合适的接收者。

### 4.4 分层知识库（Memory）

参考 Molecule AI 的 LOCAL/TEAM/GLOBAL 三层设计：

```
knowledge/
  global/              # 全局知识（所有 Agent 可读）
    coding-standards.md
    architecture-decisions.md
  teams/
    engineering/       # 团队知识
      api-design-guide.md
      deployment-runbook.md
  agents/
    dev-claude/        # Agent 私有记忆
      lessons-learned.md
      useful-commands.md
```

Agent 通过 `crewden_search_knowledge --scope global|team|personal` 搜索知识，写入时指定 scope。

**记忆晋升为技能**的可选机制：当某个知识片段被反复使用和验证后，可以手动或半自动地晋升为 Skill。

### 4.5 Crewden 改造影响

| 改动                  | 范围                      | 工作量 |
| --------------------- | ------------------------- | ------ |
| Inbox 确认闸门        | Crewden Store + Task 服务 | 1 周   |
| Agent Card 自动生成   | PaseoRuntimeService       | 3 天   |
| 分层知识库模型 + CRUD | Crewden Store + 工具集    | 1 周   |
| 记忆晋升机制          | 可选模块                  | 3 天   |

---

## 五、路线四：Dispatcher 驱动的任务引擎（Hermes Kanban 启发）

**核心思想**：引入 **中央 Dispatcher** 作为任务调度器。使用 CAS + TTL 机制确保任务领取的安全性，杜绝"一个任务两个 Agent 执行"的问题。

### 5.1 Hermes 设计的精华

Hermes Kanban 的核心机制——SQLite `BEGIN IMMEDIATE` + CAS（Compare-and-Swap）原子领取：

```python
# 核心 SQL：只有状态='ready' 且 未被领取的任务才能被 claim
UPDATE tasks
   SET status = 'running', claim_lock = ?, claim_expires = ?
 WHERE id = ?
   AND status = 'ready'       # ← 确保状态正确
   AND claim_lock IS NULL      # ← 确保未被他人领取
```

配合三层回收机制：

1. **TTL 过期回收**：`claim_expires < now()` 的任务自动 reclaim 回 `ready`
2. **进程崩溃检测**：扫描本机 PID 是否存活，不存活则 reclaim
3. **spawn 失败限制**：连续 5 次 spawn 失败 → 自动 block，防止死循环

### 5.2 7 列看板状态

```
triage → todo → ready → running → blocked → done → archived
```

| 列           | 含义                                 | Agent 视角                           |
| ------------ | ------------------------------------ | ------------------------------------ |
| **triage**   | 原始想法，等待 specifier 补充规格    | "这个需求不清晰，等人补充"           |
| **todo**     | 已定稿但未就绪（依赖未满足或未分配） | "我有了说明，但还不能做"             |
| **ready**    | 已分配且可执行                       | 🔴 Dispatcher 请注意：我可以被领取了 |
| **running**  | 正在执行（已被某个 Agent 领取）      | "我正在做这个，有 claim 锁"          |
| **blocked**  | 被阻塞（等待人工确认或外部事件）     | "我需要帮助"                         |
| **done**     | 完成                                 | "做好了"                             |
| **archived** | 归档                                 | "历史记录"                           |

### 5.3 Run 历史记录（task_runs）

每次执行尝试都是一条独立记录：

```typescript
interface TaskRun {
  id: number;
  taskId: string;
  profile: string; // 执行者
  outcome:
    | "completed"
    | "blocked"
    | "crashed"
    | "timed_out"
    | "spawn_failed"
    | "gave_up"
    | "reclaimed";
  summary?: string; // 结构化交付摘要
  metadata?: object; // 结构化交付数据（如 {"changed_files":[...]}）
  error?: string;
  startedAt: number;
  endedAt?: number;
}
```

这保证了"犯错有记录、失败有原因、下次有参考"。

### 5.4 build_worker_context 的层次化上下文组装

Agent 启动时看到的不是简单 prompt，而是分层组装：

```
1. Task title（必须）
2. Task body（开场描述，上限 8KB）
3. 本任务的历史尝试（最近 N 次，含 outcome + error + metadata）
4. 已完成父任务的 handoff 结果（summary + metadata）
5. 该 assignee 最近完成的跨任务角色历史（最近 5 个 completed run）
6. 评论线程（最近 N 条）

所有层都有字符上限，防止 prompt 膨胀
```

### 5.5 对 Crewden 的改造建议

Paseo 的 Agent 是持久会话（turn-based），不是每次 spawn 子进程。所以不能完全照搬 spawn 模式，但可以借鉴 dispatcher 的**调度决策层**：

```
PaseoRuntimeService 增强版：
  ┌─ Dispatcher（调度决策）────────────────┐
  │  1. 扫描所有 Agent 的 sessionStatus    │
  │  2. 回收 stale running 任务             │
  │  3. 根据 depends_on 推进 ready 任务     │
  │  4. 选择最优 idle Agent → 投递任务      │
  │  5. 监控 claim TTL → 超时回收           │
  └──────────────────────────────────────────┘
        ↓ 通过 PaseoDaemonClient.sendMessage
  ┌─ Agent（执行）──────────────────────────┐
  │  保持 Paseo 持久会话                    │
  │  收到任务 → turn 执行 → 完成/失败       │
  └──────────────────────────────────────────┘
```

### 5.6 Crewden 改造影响

| 改动                          | 范围                             | 工作量 |
| ----------------------------- | -------------------------------- | ------ |
| Dispatcher 调度器             | 新服务（PaseoRuntimeService 层） | 1 周   |
| CAS Claim + TTL + 心跳续期    | Crewden Store + Task 服务        | 3 天   |
| 任务领取/释放/回收 API        | Crewden REST API                 | 2 天   |
| build_worker_context 等价实现 | PaseoRuntimeService              | 1 周   |
| Run 历史记录表 + UI           | Crewden Store + Web              | 1 周   |
| Spawn 失败治理 + auto-block   | Task 服务                        | 2 天   |

---

## 六、路线五：Meta-Agent 自治理（Hermes Curator 启发）

**核心思想**：当前所有方案都是"人管理 Agent 的工具"。这条路线是"Agent 管理 Agent 的工具"。引入专门的 Meta-Agent（类似 Hermes 的 Curator），负责组织层面的自我维护。

### 6.1 Hermes Curator 的原型

Hermes v0.12 引入的 Curator：

- 一个后台 Agent 按照 cron（默认 7 天）运行
- 自动评分、修剪、合并 skill 库
- 输出 `run.json` + `REPORT.md` 报告
- 有防御机制保护内置 skill 不被篡改
- 按使用频率排名 skill

这是"**Agent 管理 Agent**"的落地实现。

### 6.2 四个 Curator 的职责

#### 1. Knowledge Curator（知识库维护者）

一个专门的 Agent，按周/月运行：

- 扫描知识库 → 识别重复/冲突内容 → 建议合并
- 识别长期未被引用的知识 → 建议归档
- 识别被频繁引用的知识 → 标记为"已验证"
- 输出 Knowledge Health Report

#### 2. Task Auditor（任务审计者）

周期性扫描任务状态：

- 检测长期 stuck 在 `starting` 的任务（对应 turn-based 问题）
- 检测连续失败的 Agent → 建议人工干预
- 检测依赖环路 → 发出警告
- 输出 Task Health Report

#### 3. Org Optimizer（组织优化者）

周期性分析 Agent 负载和效率：

- 识别过载 Agent → 建议重新分配
- 识别空闲 Agent → 建议扩展职责范围
- 分析任务完成质量 → 建议 Role 调整
- 输出 Org Health Report

#### 4. Context Curator（上下文维护者）

管理 Agent 的 system prompt 和 skill 质量：

- 识别 prompt 中的过时信息
- 将高频成功的 Agent 行为模式晋升为 Rule/Skill
- 将不再触发的 Rule 归档

### 6.3 架构落点

```
Crewden Server 新增:
  curator/
    knowledge-curator.ts    # 知识库维护 Meta-Agent
    task-auditor.ts          # 任务审计 Meta-Agent
    org-optimizer.ts         # 组织优化 Meta-Agent

每个 Curator = 一个特殊的 Paseo Agent:
  - 拥有只读访问 Crewden Store 的权限
  - 通过 MCP 工具写入建议/报告到 Channel
  - 按 cron 或事件触发运行
  - 输出可读的报告到 Channel / Inbox
```

### 6.4 Crewden 改造影响

| 改动                                  | 范围                       | 工作量 |
| ------------------------------------- | -------------------------- | ------ |
| Curator 调度框架                      | PaseoRuntimeService 新模块 | 1 周   |
| Knowledge Curator                     | 新服务                     | 1 周   |
| Task Auditor                          | 新服务                     | 3 天   |
| Org Optimizer                         | 新服务                     | 3 天   |
| Curator 权限控制（只读 + 受限工具集） | 安全层                     | 3 天   |

---

## 七、路线六：多态 Actor 社交协作模型（Multica 启发）

**核心思想**：将 Crewden 的所有协作对象从"人-only"升级为"人 + Agent 对等"。Agent 在协作图谱中拥有和人完全相同的操作能力。

### 7.1 当前 Crewden 的问题

Crewden 当前的 Message 模型假设发送者是人：

```typescript
// 当前：Message 的 author 是 user
interface Message {
  id: string;
  channelId: string;
  authorId: string; // ← 只有 human user id
  content: string;
  createdAt: Date;
}
```

Agent 的回复是通过 Paseo Timeline 转发到 Channel 的特殊处理——Agent 不是"消息的作者"，而是"另一个数据源"。

### 7.2 改为 Polymorphic Actor

```typescript
// 改造后：所有"谁做了什么"都是多态 actor
interface Message {
  id: string;
  channelId: string;
  actorType: "human" | "agent" | "system";
  actorId: string;
  actorName: string; // 展示名
  actorAvatar?: string; // agent 自动生成头像
  content: string;
  createdAt: Date;
}

interface Task {
  id: string;
  creatorType: "human" | "agent"; // ← Agent 也能创建 Task
  creatorId: string;
  assigneeType: "human" | "agent"; // ← 可以分配给 Agent
  assigneeId: string;
}

interface Comment {
  id: string;
  targetType: "task" | "message" | "channel";
  targetId: string;
  actorType: "human" | "agent";
  actorId: string;
  content: string;
}

// Agent 也有 inbox！
interface InboxItem {
  recipientType: "human" | "agent";
  recipientId: string;
  type: "assigned" | "mentioned" | "subscribed" | "task_completed";
  read: boolean;
}
```

### 7.3 解锁的能力

**1. Agent @ Agent 触发协作**

```
PM Agent 在 Task 评论中：
  "@dev-agent 请处理这个安全问题，参考 Q2 OKR KR-03 的 SLA 要求"

结果：
  → dev-agent 被自动订阅到这个 task
  → dev-agent 的 inbox 收到通知
  → 如果配置了 autopilot：自动创建子任务指派给 dev-agent
```

**2. Agent 可以主动创建 Task**

```
Task Board 上的每一张卡：
  Created by: [🤖 PM-Agent] · Assigned to: [🤖 Dev-Agent]
  ── 没有"人"参与创建，两个 Agent 自主完成工作链
```

**3. Agent 有 Inbox**

```
Dev-Agent 的 inbox：
  📬 @ 提到 (2)
  📋 新任务分配 (1)
  🔔 订阅的 task 状态变更 (3)

Agent 在下一轮 turn 开始时：
  → 优先处理 inbox 中的 @ 和分配
  → 然后处理队列中的 pending tasks
```

**4. 统一的 Activity Feed**

```
Timeline（所有 actor 混合展示）:
  10:30  🤖 QA-Agent created task "Review PR #42"
  10:32  🤖 QA-Agent commented on "Review PR #42": "发现潜在 SQL 注入"
  10:33  👤 lijun assigned task to 🤖 Dev-Agent
  10:35  🤖 Dev-Agent changed task status to in_progress
  10:42  🤖 Dev-Agent completed task "Fix SQL injection in PR #42"
  10:43  🤖 Dev-Agent created task "Re-review PR #42" → assigned to 🤖 QA-Agent
```

### 7.4 与 Multica 的差异

| 维度           | Multica                            | Crewden 改造版                    |
| -------------- | ---------------------------------- | --------------------------------- |
| Actor 类型     | member / agent / system            | human / agent / system            |
| 适用范围       | Issue + Comment + Activity + Inbox | Message + Task + Comment + Inbox  |
| Agent Inbox    | 有（收通知）                       | 有（收通知 + 确认请求）           |
| @mention 触发  | 自动创建 task                      | 自动创建 task + 绑定到 Paseo 会话 |
| Agent 创建资源 | Issue + Comment                    | Task + Message + Comment          |

### 7.5 Crewden 改造影响

| 改动                                       | 范围                        | 工作量 |
| ------------------------------------------ | --------------------------- | ------ |
| Message/Task/Comment 模型加 actor_type     | Crewden Store + shared 类型 | 1 周   |
| Agent Inbox 模型 + 通知推送                | Crewden Store + WebSocket   | 1 周   |
| @mention 自动触发 task                     | Crewden API + Task 服务     | 3 天   |
| Activity Feed 统一展示                     | Crewden Web                 | 1 周   |
| 数据迁移（现有消息转为 actorType='human'） | 迁移脚本                    | 1 天   |

---

## 八、路线七：Autopilot 自主工作引擎（Multica 启发）

**核心思想**：不依赖 agent 自己的 heartbeat（agencycli 模式），也不依赖中央 dispatcher 定时扫描（Hermes 模式），而是用**声明式规则**定义 agent 何时工作、以什么形式工作。

### 8.1 三种调度模式对比

| 模式                          | 来源      | 适合场景                   |
| ----------------------------- | --------- | -------------------------- |
| **Heartbeat**（agent 自驱动） | agencycli | Agent 持续运营、监控类工作 |
| **Dispatcher**（中央分配）    | Hermes    | 任务池大、多 agent 抢任务  |
| **Autopilot**（规则触发）     | Multica   | 定时例行工作、事件驱动工作 |

三个模式**不互斥**——可以并存：

```
Crewden 完整调度体系：
  ├── Autopilot 层（何时触发）
  │     ├── Schedule:  daily bug triage, weekly report
  │     ├── Webhook:   GitHub PR opened → create review task
  │     ├── Event:     new issue labeled "urgent" → dispatch immediately
  │     └── API:       manual "run now" button
  │
  ├── Dispatcher 层（谁执行 —— 对应 Hermes）
  │     ├── Claim 机制: CAS + TTL + heartbeat 续期
  │     ├── 负载均衡:  选择最空闲的 agent
  │     └── 故障恢复:  超时 reclaim → 重新分配
  │
  └── Heartbeat 层（agent 空闲时做什么 —— 对应 agencycli）
        ├── 检查 inbox
        ├── 执行 pending 队列
        ├── 执行 wakeup.md（主动找活）
        └── 更新自身状态
```

### 8.2 Autopilot 数据模型

```typescript
interface Autopilot {
  id: string;
  workspaceId: string;
  title: string;
  description?: string;

  // 执行模式
  executionMode: "create_task" | "send_message" | "run_prompt";
  // create_task: 创建 task 到 Task Board → 正常流程
  // send_message: 发送消息到 Channel → Agent 直接回复
  // run_prompt: 直接执行 prompt → 结果写入 log

  // 分配给谁
  assigneeType: "agent" | "role" | "any"; // any = 自动选最优
  assigneeId?: string;
  roleId?: string; // 分配给某个 role 的任意 agent

  // 任务模板
  taskTemplate?: {
    title: string; // 支持 {{date}} {{trigger_source}}
    body?: string;
    priority: number;
    channelId?: string; // 在哪个 channel 创建
    parentTaskId?: string; // 子任务模板
  };

  // 触发器（可多个）
  triggers: AutopilotTrigger[];

  // 并发策略
  concurrencyPolicy: "skip" | "queue" | "replace" | "parallel";

  // 运行时状态
  enabled: boolean;
  lastRunAt?: Date;
  nextRunAt?: Date;
}

interface AutopilotTrigger {
  id: string;
  kind: "schedule" | "webhook" | "event" | "api";

  // schedule: cron + timezone
  cronExpression?: string; // "0 9 * * 1-5"
  timezone?: string; // "Asia/Shanghai"

  // webhook
  webhookUrl?: string;
  webhookSecret?: string;

  // event
  eventType?: string; // "task.completed" | "message.created" | "agent.idle"
  eventFilter?: object; // { "channelId": "xxx", "priority": "critical" }
}
```

### 8.3 内置 Autopilot 模板

```
📋 Bug Triage         — 每个工作日早上 9 点，扫描 issues 创建 triage tasks
🔍 PR Review Reminder — 每个工作日早上 10 点，检查 open PR 创建 review tasks
📊 Weekly Report      — 每周五下午 5 点，汇总本周工作生成报告
🔄 Dependency Audit   — 每周一早上 10 点，审计依赖更新
🛡️ Security Scan     — 每周日凌晨 2 点，运行安全扫描
💬 Standup Reminder   — 每个工作日上午 9:30，发送 standup 提醒到 channel
```

### 8.4 与 Paseo Turn-based 模型的协同

Autopilot 在触发时创建一个 **新的 Paseo turn**，这与 Paseo 的 turn-based 模型完全兼容：

```
Autopilot 触发
  → executionMode = 'create_task'
    → 创建 Task（creatorType = 'system', originType = 'autopilot'）
    → assign 到目标 agent
    → agent 下一个 turn 时处理
  → executionMode = 'send_message'
    → 发送消息到 channel
    → @agent 触发 inbox 通知
    → agent 下一个 turn 时回复
  → executionMode = 'run_prompt'
    → 直接通过 PaseoDaemonClient.sendMessage 投递 prompt
    → agent 立即执行（如果 idle）
    → 结果写入 autopilot_run 记录
```

### 8.5 Crewden 改造影响

| 改动                        | 范围                | 工作量 |
| --------------------------- | ------------------- | ------ |
| Autopilot 模型 + CRUD       | Crewden Store + API | 1 周   |
| Cron 调度器（每 30s 扫）    | 新后台服务          | 2 天   |
| Webhook 端点 + 安全验证     | Crewden REST API    | 2 天   |
| Autopilot 运行记录 + 状态机 | Crewden Store       | 3 天   |
| Autopilot UI 管理页面       | Crewden Web         | 1 周   |
| 内置模板 + 一键创建         | Crewden Web         | 2 天   |

---

## 九、路线八：Skill 作为上下文注入的原生机制（Multica 启发）

**核心思想**：借鉴 Multica 的 Skill 设计——Skill 不是代码、不是 prompt 模板，而是**纯 Markdown 文件**。Daemon/Server 在启动 Agent 时注入到工作目录的 provider 原生路径。Agent CLI 按自己的约定发现并读取。

### 9.1 Multica 的 Skill 注入机制

```
Skill "react-patterns":
  ├── SKILL.md          # 主文档
  ├── examples/hooks.md
  └── examples/patterns/useCallback.md

Daemon 注入（每个 provider 路径不同）:
  Claude Code:  .claude/skills/react-patterns/SKILL.md
  Codex:        CODEX_HOME/skills/react-patterns/
  OpenCode:     .config/opencode/skills/react-patterns/SKILL.md
  Pi:           .pi/agent/skills/react-patterns/SKILL.md
  Cursor:       .cursor/skills/react-patterns/SKILL.md
```

### 9.2 三层上下文体系

```
Crewden Agent 的三层上下文:
  ┌─ Role Context ──────────────────────────────┐
  │  你是谁（system prompt 中的角色定义）         │
  │  例: "你是一个资深前端工程师..."              │
  └──────────────────────────────────────────────┘
  ┌─ Skill Context ─────────────────────────────┐
  │  怎么做（领域知识 Markdown 文档）             │
  │  例: 部署流程 SOP、代码审查清单、UI 设计规范  │
  └──────────────────────────────────────────────┘
  ┌─ Task Context ──────────────────────────────┐
  │  这次做什么（具体的 task prompt）             │
  │  例: "Review PR #42..."                      │
  └──────────────────────────────────────────────┘
```

### 9.3 与现有方案的对比

| 维度         | Multica Skill                 | agencycli Skill                         | Hermes Skill               |
| ------------ | ----------------------------- | --------------------------------------- | -------------------------- |
| **格式**     | Markdown + 附加文件           | SKILL.md（YAML frontmatter + Markdown） | SKILL.md + references      |
| **注入方式** | Daemon 写入 provider 原生路径 | sync 时合并到 agent context             | --skills 命令行参数        |
| **发现**     | 支持从 URL 导入               | CLI install / hub                       | 自动发现本地路径           |
| **进化**     | 无（纯静态）                  | 路线图中（memory → skill promotion）    | Curator 自动评分/修剪/合并 |

### 9.4 Skill 生命周期（进化路径）

```
1. Memory → 反复使用的知识片段（Agent 私有）
2. Draft Skill → 经过 curator 评审的模式 → 提升为 Skill
3. Verified Skill → 被多次成功执行验证的 Skill
4. Deprecated Skill → 长期未使用 → Curator 归档
```

### 9.5 Crewden 改造影响

| 改动                                    | 范围                 | 工作量 |
| --------------------------------------- | -------------------- | ------ |
| Skill 模型 + CRUD                       | Crewden Store + API  | 3 天   |
| Daemon 注入适配（多 provider 路径映射） | PaseoRuntimeService  | 1 周   |
| Skill 与 Agent/Channel 的关联管理       | Crewden API + Web UI | 3 天   |
| Skill 导入（URL/github/手动）           | Crewden API          | 2 天   |
| Skill 生命周期 + 进化                   | 可选模块             | 1 周   |

---

## 十、八条路线融合总图

```
                        ┌──────────────────────────────────────────┐
                        │   路线六：Polymorphic Actor 社交图谱       │
                        │   Agent = 人（同等操作权限）               │
                        │   @mention / Subscribe / Reaction / Inbox │
                        └────────────────┬─────────────────────────┘
                                         │ 建立在
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                          │
              ▼                          ▼                          ▼
    ┌─────────────────┐      ┌───────────────────┐      ┌───────────────────┐
    │ 路线七：Autopilot│      │ 路线一：Org+      │      │ 路线三：沟通+     │
    │ 自主工作引擎     │      │ Scheduler         │      │ 知识库            │
    │ (Multica 启发)  │      │ (agencycli 启发)  │      │ (agencycli+启发)  │
    └────────┬────────┘      └────────┬──────────┘      └────────┬──────────┘
             │                        │                          │
             │     ┌──────────────────┴──────────────────┐       │
             │     │  路线四：Dispatcher（Hermes 启发）  │       │
             │     │  Claim + TTL + Spawn 治理           │       │
             │     └──────────────────┬──────────────────┘       │
             │                        │                          │
             └────────────────────────┼──────────────────────────┘
                                      │
                          ┌───────────┴───────────┐
                          │  路线八：Skill 注入    │
                          │  原生路径上下文注入    │
                          │  (Multica 启发)       │
                          └───────────────────────┘
                                      │
                          ┌───────────┴───────────┐
                          │  路线二：OKR/目标      │
                          │  路线五：Curator 自治理│
                          └───────────────────────┘
```

### 八条路线的互补关系

| 路线       | 核心方向                  | 灵感来源             | 抽象层级                       |
| ---------- | ------------------------- | -------------------- | ------------------------------ |
| **路线一** | 组织架构 + Heartbeat 调度 | agencycli            | 战略层（谁是员工，谁做什么）   |
| **路线二** | Task 治理 + OKR/Milestone | agencycli            | 战略层（为什么做，进度到哪里） |
| **路线三** | 沟通 + Inbox + 知识库     | agencycli + Molecule | 协同层（Agent 之间怎么沟通）   |
| **路线四** | Dispatcher + CAS Claim    | Hermes Kanban        | 调度层（任务怎么安全分配）     |
| **路线五** | Meta-Agent Curator        | Hermes Curator       | 自治层（Agent 管理 Agent）     |
| **路线六** | Polymorphic Actor         | Multica              | 基础层（Agent 和人等权）       |
| **路线七** | Autopilot 规则触发        | Multica              | 调度层（声明式自动化）         |
| **路线八** | Skill 上下文注入          | Multica              | 执行层（Agent 的知识输入）     |

### 设计哲学演进

```
      agencycli:             Hermes:                  Multica:
    "公司管理"             "工厂调度"               "社交协作"
         │                     │                        │
         ▼                     ▼                        ▼
    ┌────────┐          ┌──────────────┐         ┌────────────┐
    │员工模型│          │Dispatcher    │         │一等公民    │
    │Heartbeat│         │CAS Claim     │         │@mention    │
    │OKR     │          │Run History   │         │Autopilot   │
    │Inbox   │          │TTL 安全网    │         │Skill 注入  │
    │Context │          │Curator       │         │会话恢复    │
    │Grid    │          │              │         │            │
    └────────┘          └──────────────┘         └────────────┘
          │                    │                        │
          └────────────────────┼────────────────────────┘
                               │
                               ▼
                    Crewden 融合方案
          ┌──────────────────────────────────────────────┐
          │  路线六（基础数据层）：Polymorphic Actor     │
          │  路线四（调度安全层）：Dispatcher Claim      │
          │  路线一/七（工作触发层）：Heartbeat + Autopilot│
          │  路线八（知识输入层）：Skill 注入             │
          │  路线二（目标方向层）：OKR + Milestone       │
          │  路线三（沟通协同层）：Inbox + 知识库        │
          │  路线五（自进化层）：Curator                 │
          └──────────────────────────────────────────────┘
```

---

## 十一、实施优先级建议

### 11.1 依赖关系图

```
路线六 (Polymorphic Actor) ─── 无前置依赖，但其他路线依赖它
      │
      ├──→ 路线一/三/四/七/八 都依赖 Agent 是"一等公民"的数据模型
      │
路线四 (Dispatcher) ─── 需要正确的状态机，但可以作为路线二的前置
      │
      ├──→ 路线二 (Task 治理) 需要 Dispatcher 提供的 Claim/Run 机制
      │
路线一 (Org + Heartbeat) ─── 可以与路线四并行
路线七 (Autopilot) ─── 可以与路线四并行
路线八 (Skill) ─── 无前置依赖
      │
      ├──→ 路线五 (Curator) 需要路线三的知识库 + 路线八的 Skill
      │
路线三 (沟通 + 知识库) ─── 部分依赖路线六
路线二 (OKR/Milestone) ─── 需求路线四的 Run 数据 + 路线六的 Actor 模型
```

### 11.2 推荐实施顺序

| 优先级 | 路线                 | 内容                                          | 工作量 | 前置                 |
| ------ | -------------------- | --------------------------------------------- | ------ | -------------------- |
| **P0** | 路线六（最小可行版） | Message + Task 加 `actor_type`，让 Agent 可见 | 1 周   | 无                   |
| **P0** | 现有 Turn-based 根治 | 三层状态机拆分（已有方案）                    | 1-2 周 | 无                   |
| **P0** | 路线四（核心）       | Dispatcher Claim + TTL + Run 记录             | 1-2 周 | P0 状态机            |
| **P1** | 路线二（部分）       | Task 7 态 + DAG + Triage                      | 2 周   | P0 Dispatcher        |
| **P1** | 路线一（最小可行版） | Org + Role 上下文注入                         | 2 周   | P0 Polymorphic Actor |
| **P2** | 路线七               | Autopilot 引擎 + 内置模板                     | 2 周   | P0 Dispatcher        |
| **P2** | 路线八               | Skill 模型 + 注入                             | 1-2 周 | 无                   |
| **P2** | 路线一（扩展）       | Heartbeat + wakeup.md                         | 2 周   | P1 Org               |
| **P3** | 路线二（扩展）       | OKR + Milestone 目标系统                      | 2 周   | P1 Task              |
| **P3** | 路线三               | Inbox 闸门 + 分层知识库                       | 2-3 周 | P0 Actor             |
| **P4** | 路线五               | Curator Meta-Agent                            | 2 周   | P2 Skill + P3 知识库 |

### 11.3 起步建议（最小闭环）

如果要从最轻量开始，建议先做三条路线的最小可行版本：

```
第一步（1-2 周）：路线六核心
  → Message 和 Task 增加 actor_type
  → Agent 作为 creator/commenter 可见
  → 现有数据迁移

第二步（1 周）：路线四核心
  → Dispatcher 扫描 idle Agent
  → CAS 原子领取 + TTL 回收
  → 基础的 Run 记录

第三步（2 周）：路线二核心
  → Task 状态机从 3 态到 7 态
  → task_links 依赖图
  → Triage 列（"原始想法"先入 triage）
  → awaiting_confirmation 确认闸门

完成这三步后，Agent 协作层的基础就具备了：
  ✓ Agent 和人平等（路线六）
  ✓ 任务领取安全（路线四）
  ✓ 任务生命周期完整（路线二）
```
