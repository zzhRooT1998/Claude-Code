# Claude Code 上下文压缩机制

> 学习笔记。两大类：full compact（有损整体摘要）+ micro compact（轻量瘦身）。
> 核心源码：`src/services/compact/`、`src/query.ts`

---

## 0. 总览：两类压缩

| | full compact | micro compact |
|--|--|--|
| 操作 | 把全部消息摘要成九段，替换历史（**有损**） | 删旧 tool result / thinking（**可恢复**） |
| 定位 | 撞顶兜底，最重 | 连续瘦身，跑在前面 |
| 触发轴 | token（167k） | 时间 / 数量 / token，分三种方式 |

**级联设计**：micro 持续把曲线压低（可恢复，只动 tool result），full compact 是「micro 清无可清、非工具内容撑满」时的最后手段（有损）。

---

## 1. 每轮 loop 的检查顺序

主循环 `while(true)`（`query.ts:307`）每迭代一次（= 每个 API turn），开头按顺序过：

```
每轮 loop:
  ① microcompact (query.ts:414)  ← 查 cached triggerThreshold / time-based gap
  ② autocompact                   ← 查 167k token 阈值 (full compact 入口)
  ③ API 调用                      ← 带 context_management 策略, 服务端查 180k
```

三道都是「每轮检查」，只是查的轴不同（数量 / token / 时间），且 cached 那道外部版被 DCE。

---

## 2. Full compact（有损整体摘要）

### 触发（三个入口，都通向同一套 `compactConversation`）

| 入口 | 触发 |
|------|------|
| **autocompact** | 每轮 token ≥ 阈值（自动） |
| 手动 /compact | 用户输入 |
| reactive | 模型返回上下文溢出报错（兜底） |

> autocompact 不是「另一种 compact」，是 full compact 的**自动触发器**。

### 阈值（`autoCompact.ts`）

```
阈值 = effectiveWindow − 13_000               (AUTOCOMPACT_BUFFER_TOKENS)
effectiveWindow = contextWindow − min(maxOutput, 20_000)
```

| 窗口 | effectiveWindow | full compact 阈值 |
|------|----------------|------------------|
| 200k | 180k | **167k**（≈effective 的 93%） |
| 1M | 980k | 967k |

两层 buffer：20k 给「摘要输出留空间」，13k 是「触发提前量」（在撑爆前就压）。

### 逻辑

从上一压缩边界到最近消息 → 生成**九段摘要** → 插入压缩边界 + 摘要 + attachment：

```
九段摘要: Primary Request / Key Technical Concepts / Files and Code /
          Errors and fixes / Problem Solving / All user messages /
          Pending Tasks / Current Work / Optional Next Step

最终消息顺序:
  boundaryMarker(SystemMessage)
  → summaryMessages(UserMessage, isCompactSummary)
  → messagesToKeep(部分压缩才有)
  → attachments(最近5个文件 + plan + skill + tool/agent/mcp delta)
  → hookResults
```

`POST_COMPACT_MAX_FILES_TO_RESTORE = 5`。压缩后会 PreCompact/PostCompact hook + SessionStart hook。

---

## 3. Micro compact（轻量瘦身，三种方式）

| # | 方式 | 怎么删 | 触发轴 | 门控 / 对外开放 |
|---|------|--------|--------|----------------|
| 1 | **time-based** | 客户端改本地（旧 tool result → `[Old tool result content cleared]`） | **时间**：距上条 assistant > 60min | GrowthBook `tengu_slate_heron`，默认关（可远程开 → 外部可能能用） |
| 2 | **cached** | 客户端发 `cache_edits`（挑 id）→ 服务端删 KV 缓存 | **数量**：可压缩 tool result 数 > triggerThreshold，保留 keepRecent | `feature('CACHED_MICROCOMPACT')` ant-only（外部 DCE）❌ |
| 3 | **API context_management** | 声明式策略，服务端按阈值删 | **token**：input_tokens ≥ 180k | tool 清理 ant-only+env ❌；**thinking 清理对外部开放 ✅** |

旧 legacy 路径已删除（`tengu_cache_plum_violet` 恒 true）。

### 三种的触发轴不同（重点）

```
① time-based       → 时间 (60min gap, 缓存冷)
② cached           → 数量 (tool result 个数超阈值)  ← 持续压曲线、跑在 full compact 前
③ context_management → token (180k, 服务端判)
```

### 短路顺序

`microcompactMessages` 内：**time-based 先判，命中（60min gap）就短路**，缓存冷了不走 cached。cached 只在缓存还热时跑。

### 「可压缩工具」白名单
Read / Shell / Grep / Glob / WebSearch / WebFetch / Edit / Write —— 只清这些，其它工具（Task/MCP）结果不动。

---

## 4. cache editing 后端 API（context_management）

第 3 种用的是 **Anthropic 官方 Context Management API**（带版本号），挂在请求体：

```json
{
  "context_management": {
    "edits": [
      { "type": "clear_thinking_20251015", "keep": "all" },
      {
        "type": "clear_tool_uses_20250919",
        "trigger":        { "type": "input_tokens", "value": 180000 },
        "clear_at_least": { "type": "input_tokens", "value": 140000 },
        "clear_tool_inputs": ["Bash","Read","Grep","Glob","WebFetch","WebSearch"],
        "exclude_tools":  ["Edit","Write","NotebookEdit"],
        "keep":           { "type": "tool_uses", "value": "N" }
      }
    ]
  }
}
```

字段：
- `trigger`：触发条件（input_tokens ≥ value）
- `clear_at_least`：至少清这么多 token（180k−40k=140k）
- `clear_tool_inputs`：清哪些工具结果（数组或 true）
- `exclude_tools`：永不清的工具
- `keep`：保留最近 N 个

默认：触发 180k、保留 40k，可用 env `API_MAX_INPUT_TOKENS` / `API_TARGET_INPUT_TOKENS` 改。

### 逻辑：声明式 + 服务端确定性重放

```
客户端: 把策略(政策)挂在【每个】请求上, 不改本地, 继续发 full
服务端: 处理时评估 trigger → input_tokens ≥ 180k → 在服务端清 → 返回 cache_deleted_input_tokens
```

**一致性关键**：客户端继续发 full、不知道清了哪几条，但缓存仍命中——因为**缓存键是「清除后的状态」，而清除策略确定性幂等，每次把 full 重新清成同一个清后前缀 → 命中**。不是「full 还缓存着」，而是「服务端确定性地重清成同一态」。

> 命令式的 cache_edits（第 2 种）则靠客户端 `pinCacheEdits` 把删除指令 pin 在原位、每次重发来保证一致。
> 服务端如何在 KV 层 stitch 是 API 内部行为，不在本 repo。

---

## 5. 阈值对比：167k vs 180k 的常见误解

- **167k = full compact**（effective − 13k，客户端）
- **180k = API context_management 的默认 trigger**（= effectiveWindow，服务端，ant-only）

⚠️ **不要直接比出「full compact 比 micro 先触发」**：
- 真正持续压曲线、跑在 full compact 前的 micro 是**第 2 种（按数量连续清）**，早在 167k 之前就一直在工作，不是 180k 那条线。
- 180k 是「服务端额外清理机制」的自己的默认值，且外部版不启用，不该和 167k 直接赛跑。

### 为什么 full compact 仍是必需（micro 不够）
micro 只能清 **tool result**（可恢复、落盘），清不动 assistant 推理 / user 消息 / 对话本身。到 167k 时往往是「micro 清无可清、非工具内容撑满」——**只有 full compact（整体摘要）能回收那部分**。它有损，所以放最后。

---

## 6. 和 SessionMemory 的关系：compact 摘要的「预制版」

`trySessionMemoryCompaction` 在 `compactConversation` 之前先跑：有 SessionMemory 内容就**直接拿它当压缩产物，跳过九段摘要生成**。

- 九段摘要：压缩那一刻临时 LLM 生成（慢、阻塞）
- SessionMemory：平时增量维护好，压缩时直接拿（快）

两者内容高度重叠（都是对话精华），本质是「临时生成」vs「平时预制」。`tengu_session_memory` 默认关。

---

## 7. 设计哲学

1. **可恢复优先、有损兜底**：micro（删可重读的 tool result）连续在前，full compact（有损摘要）撞顶才上。
2. **多轴触发**：micro 三种分别按时间/数量/token 触发，覆盖不同场景（缓存冷 / 结果堆积 / token 满）。
3. **声明式 + 服务端确定性**：context_management 客户端只发政策，服务端幂等重放，缓存自然一致。
4. **大量 ant-only/实验**：cached、tool 清理、SessionMemory 都默认关或 ant-only——外部版实际生效的主要是 full compact + （开 thinking 时的）thinking 清理 + 可远程开的 time-based。
