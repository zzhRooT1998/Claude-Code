# Claude Code 多 Agent 机制总结

> 学习笔记。三种多 agent 机制：AgentTool 子代理、Coordinator 模式、Swarm/Team。
> 核心源码：`src/tools/AgentTool/`、`src/coordinator/`、`src/utils/swarm/`、`src/utils/tasks.ts`

---

## 0. 共同底座（三种都一样）

所有多 agent 本质都是 **fork 一个隔离的 query loop**：

- 入口 `runAgent()`（`src/tools/AgentTool/runAgent.ts`），`createSubagentContext` 派生隔离上下文（独立 agentId / 消息历史 / 文件缓存）。
- 子 agent 跑的是**和主 Claude 完全相同的 `query()` 引擎**，只是 system prompt / 工具 / 上下文不同。
- 子 agent 流式过程被过滤，**只把最终 assistant text 回传**（`finalizeAgentTool`）→ 主上下文只装结论。
- 三种机制的派生底座**都是 AgentTool**，区别在上面叠的层。

---

## 1. AgentTool 子代理（默认，外部版唯一开启）

| 项 | 内容 |
|----|------|
| 开关 | 默认有（system prompt 里的 `Agent` 工具） |
| 主角色 | 正常干活 + 可派生 |
| 子 agent 生命周期 | 一次性，用完即弃 |
| 触发 | 模型输出 `tool_use(Agent)`，模型决策驱动 |
| 并行 | `isConcurrencySafe=true`，一条消息多个 Agent → 并发（上限 10，`MAX_TOOL_USE_CONCURRENCY`） |
| 通信 | 单向：主→子给自包含 prompt，子→主只回最终文本 |
| 传递的上下文 | 只有 prompt + userContext(CLAUDE.md) + systemContext(git/env) + 专属 system prompt + 自己的工具池 + **空白文件缓存**；**不含父对话** |
| 能否再开子 agent | **不能**（外部版 `ALL_AGENT_DISALLOWED_TOOLS` 剔除 Agent 工具，防递归）；ant 可嵌套 |

- **内置 agent**：general-purpose、Explore（只读搜索）、Plan（架构规划）、verification、claude-code-guide、statusline-setup。
- **进度**：generator `yield`/`next` 流式推送（push-on-produce，按干活节奏）；可选每 30s 摘要（定时器，给人看）；转后台后改用 `TaskGet` 轮询。
- **transcript**：写独立 sidechain（按 agentId，`isSidechain:true`），可 resume、可审计、不污染主链。

---

## 2. Coordinator 模式（编译 + env 双开关，外部未启用）

**一句话：用工具裁剪把主 Claude「锁」成纯指挥官。底层仍是机制① 的 async subagent（`subagent_type='worker'`）。**

| 项 | 内容 |
|----|------|
| 开关 | `feature('COORDINATOR_MODE')` + `CLAUDE_CODE_COORDINATOR_MODE` env |
| 主角色 | 纯指挥，**物理上拿不到 Bash/Edit/Read** |
| coordinator 工具 | 只剩 4 个：`Agent` / `SendMessage` / `TaskStop` / `SyntheticOutput`（`COORDINATOR_MODE_ALLOWED_TOOLS`） |
| worker 工具 | 完整工具池 |
| 通信 | worker→coordinator：`<task-notification>` XML（user-role 但非对话方）；coordinator→worker：`SendMessage` 续聊 |
| 工作流 | Research(并行worker) → **Synthesis(自己做)** → Implementation(worker) → Verification(worker) |
| 并发规则 | 只读自由并行；写同组文件同时只一个；验证可与实施并行 |

- **核心铁律**：禁止「based on your findings」式甩锅，coordinator 必须自己 synthesize，prompt 必须含文件路径/行号/完成标准（worker 看不到对话）。
- **continue vs spawn**：高上下文重叠→`SendMessage` 续；低重叠→`Agent` 新开。
- **像 workflow 但不是**：四阶段是 prompt 建议（**LLM 驱动**），不是代码强制的脚本（**workflow 才是代码驱动**）。

---

## 3. Swarm / Team（ant 默认开；外部需 `--agent-teams` + GrowthBook `tengu_amber_flint`）

**一句话：用「Team≡共享TaskList」做协调中枢，多个 Claude 组成对等长存活团队。**

| 项 | 内容 |
|----|------|
| 核心抽象 | **Team = TaskList（1:1）**。team 配置 `~/.claude/teams/{name}/config.json`，任务清单 `~/.claude/tasks/{name}/`（每任务一个 `{id}.json`） |
| 主角色 | `team-lead`（发起 session，`isLeader=!agentId`），对等关系 |
| 执行后端 | `tmux` / `iterm2` pane（**独立 Claude 进程**）或 `in-process`（同进程协程）；无 tmux/iTerm 退回 in-process |
| 派生 | Agent 工具 + `team_name`/`name` 参数 |
| 生命周期 | 长存活，turn 间自动 **idle**（≠下班，发消息可唤醒），可重连 |
| 通信 | `SendMessage` 按 **name** 寻址，**对等双向**（含 peer DM）；inbox + 轮询投递成对话 turn；非 A2A，是自研 mailbox（另支持 `uds:`/`bridge:`） |
| 协调 | 共享 task list **抢占式 work-stealing**：认领 pending+无owner+blockedBy空 的任务，小 ID 优先 |
| 关闭 | `SendMessage({type:"shutdown_request"})`；**leader 崩 = 团队解散**（杀孤儿 pane + 删目录），**不选主** |

**Task 数据结构**（`TaskSchema`，编码三层信息）：

- 进度：`status`（pending/in_progress/completed/failed/killed）
- 分工：`owner`（teammate name，last-write-wins，**无互斥锁**）
- 顺序：`blocks`/`blockedBy`（真 DAG，已完成 blocker 实时从 blockedBy 剔除→自动解锁）
- **执行是协作式的**：依赖/并发都建模了，但代码几乎不强制（唯一硬闸是 TaskCompleted hook），靠 agent 读 TaskList + 守 prompt 纪律自我协调。

磁盘上单个任务示例 `2.json`：

```json
{
  "id": "2",
  "subject": "实现 POST /api/login 接口",
  "description": "在 src/auth/login.ts 新增登录接口...",
  "activeForm": "实现登录接口",
  "owner": "coder",
  "status": "in_progress",
  "blocks": ["3"],
  "blockedBy": ["1"],
  "metadata": {}
}
```

TaskList 工具渲染给模型看的文本：

```
#1 [completed] 调研现有 auth 模块 (researcher)
#2 [in_progress] 实现 POST /api/login 接口 (coder)
#3 [pending] 给登录接口写集成测试 [blocked by #2]
#4 [pending] 更新前端登录表单
```

---

## 4. 触发 / 启用方式（两层：启用 vs 触发）

> 关键区分：**「启用」= 这套机制能不能用（开关）**；**「触发」= 用的时候由谁发起（运行时）**。

### ① AgentTool 子代理
- **启用**：无需配置，默认就在工具池里。
- **触发**：
  1. **模型主动调** `tool_use(Agent)` —— 模型决策驱动（主路径）。模型自己决定何时派、派哪个 `subagent_type`、并行几个。
  2. **编程式入口**（非模型）：slash command / workflow 子任务 / coordinator/swarm 派 worker，都会直接调 `runAgent()`。
- **特征**：模型驱动，不是阈值/规则驱动（对比 compact 是阈值驱动）。

### ② Coordinator 模式
- **启用**：编译期 `feature('COORDINATOR_MODE')` **且** 运行期 `CLAUDE_CODE_COORDINATOR_MODE` env 为真，两者都满足（`isCoordinatorMode()`）。外部版编译期就没开。
  - resume 旧会话时 `matchSessionMode()` 会翻转 env 去匹配会话原始模式。
- **触发**（模式内）：和① 一样靠模型调 `Agent` 工具派 worker，但被 coordinator system prompt 强约束进「研究→综合→实施→验证」四阶段。`SendMessage` 续聊已有 worker，`TaskStop` 停掉方向错的。
- **特征**：启用是 env 级；触发仍是模型驱动，只是工具被砍到 4 个、行为被 prompt 规范。

### ③ Swarm / Team
- **启用**：`isAgentSwarmsEnabled()`
  - **ant**：永远开。
  - **外部**：需要 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` env **或** `--agent-teams` flag，**再** 过 GrowthBook killswitch `tengu_amber_flint`。
- **触发**：
  1. 模型调 `TeamCreate` 工具建队（它的 prompt 要求「proactively」——用户提到 team/swarm/协作，或任务复杂到值得并行时，模型就主动建队）。
  2. 用户显式要求「组个团队 / 用 swarm」。
  3. 建队后用 `Agent` 工具 + `team_name`/`name` 派 teammate；teammate 自己读 TaskList 抢占任务。
- **特征**：启用是 ant/flag/killswitch 三重；触发是 TeamCreate（模型主动或用户显式）。

### 触发方式速查表

| 机制 | 启用条件 | 运行时触发 | 驱动者 |
|------|---------|-----------|--------|
| ① Subagent | 默认 | `tool_use(Agent)` / 编程式 `runAgent()` | 模型决策 |
| ② Coordinator | 编译 flag + env 双开关 | 模型调 Agent 派 worker（受四阶段 prompt 约束） | 模型决策 |
| ③ Swarm/Team | ant 默认 / 外部 `--agent-teams`+killswitch | `TeamCreate` 建队 + Agent 派 teammate | 模型主动 or 用户显式 |

---

## 5. 三种对照总表

| 维度 | ① Subagent | ② Coordinator | ③ Swarm/Team |
|------|-----------|---------------|--------------|
| 默认可用 | ✅ | ❌ 双开关 | ❌ ant/实验 |
| 派生底座 | AgentTool | AgentTool(worker) | AgentTool(+team) |
| 主 Claude | 干活 | 纯指挥 | team-lead 对等 |
| agent 生命周期 | 一次性 | 一次性 | **长存活/idle唤醒** |
| 通信 | 单向回文本 | worker→XML / lead→SendMsg | **对等双向 SendMessage** |
| 协调方式 | 无 | 中心派发 | **共享 TaskList 抢占** |
| 运行位置 | 同进程 fork | 同进程 fork | **独立进程(pane) 或 in-process** |
| 拓扑 | 主→叶（2层） | 星型，指挥居中 | 星型 + peer 边，leader 单点 |

---

## 6. 设计哲学（一句话串起来）

**所有多 agent = 「fork 隔离 query loop + 只回结论不回过程」**，三种区别只在：

- 主 agent 干不干活（①干 / ②不干）
- 子 agent 活多久（①②一次性 / ③长存活）
- 通信单向还是双向（①②单向 / ③双向）
- 协调靠不靠共享状态（③才有 TaskList）

共同收益是**上下文隔离**——子 agent 烧的 token、读的文件、试错都不进主上下文。控制权始终在模型手里（结构由数据/prompt 表达，执行靠 LLM 协作，代码极少硬编排）。
