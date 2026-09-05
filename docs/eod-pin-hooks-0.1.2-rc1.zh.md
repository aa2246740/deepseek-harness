# 下班钉选钩子对照 0.1.2-rc.1

[English](eod-pin-hooks-0.1.2-rc1.md) | 中文

仅作讨论与版本差异盘点。本页不是产品约定。它回答官方 DeepSeek Harness **0.1.2-rc.1** 是否补上了先前下班钉选卡片头脑风暴所要的生命周期钩子。

## 概述

**否。** 标签 `dsh-v0.1.2-rc.1`（发布名称 v0.1.2-rc.1）没有加入 `SessionEnd`、日历下班定时器、操作系统通知、`FileChanged` 监视，或 `TaskCompleted`。Claude Code 桥仍在解析时忽略这些事件。Cordis 插件仍可在既有 API 上实现自动草稿 + Pin/Keep。本 fork 仍停留在 `0.1.2-alpha.1`，落后该标签数百个提交。

## 目录

- [范围](#scope)
- [结论](#verdict)
- [所要钩子](#wanted-hooks)
- [0.1.2-rc.1 实际改了什么](#what-012-rc1-actually-changed)
- [不改核心的插件](#plugin-without-core)
- [仍需核心](#still-requires-core)
- [产品立场](#product-lean)
- [Dev Note](#dev-note)

-----

<a id="scope"></a>
## 范围

对照标签来自 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)（本 fork 没有对应标签）：

| 引用 | `package.json` 版本 | SHA | 角色 |
|---|---|---|---|
| `dsh-v0.1.1-rc.2` | `0.1.1-rc.2` | `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e` | 0.1.2-rc.1 官方 changelog 基线 |
| 本检出 / 头脑风暴 | `0.1.2-alpha.1` | `2c723c534aabf21517c9f3d2b7a4f5dec01c462f` | 先前下班钉选清单（[PR（Pull Request） #3](https://github.com/aa2246740/deepseek-harness/pull/3)） |
| `dsh-v0.1.2-rc.1` | `0.1.2-rc.1` | `a66e4702047846cdaa10c66c9d3df3951f5ea70d` | 所请求的升级 |
| `dsh-v0.1.3-alpha.1` | `0.1.3-alpha.1` | `d347e703908d0406b7a7ef80e3a0e594d86b2215` | 不在范围内；仅作为后续陷阱引用 |

本 fork 与上游都没有名为 `v0.1.2-rc.1` 的标签。上游的确切标签是 **`dsh-v0.1.2-rc.1`**。发布说明：[v0.1.2-rc.1](https://github.com/deepseek-ai/deepseek-harness/releases/tag/dsh-v0.1.2-rc.1)。上游在本 fork 与 rc.1 之间还打了 `dsh-v0.1.2-alpha.2` 至 `dsh-v0.1.2-alpha.5`；本 fork 没有这些标签。`CLAUDE_EVENTS` 与 `KNOWN_SESSION_EVENT_TYPES` 成员集合在这些标签上均未变。`SessionEvent.ignorable` 在 alpha.2 恢复；`Session.events` 在 alpha.4 删除。没有所要的钉选钩子出现后又消失。

头脑风暴于 2026-09-04 对本 fork 写成，晚于 0.1.2-rc.1 在 2026-09-03 发布。它描述的已是 0.1.2-alpha.1 API，而不是 0.1.1-rc.2。因此有信息量的窗口是 **alpha.1 → rc.1**，外加对官方 **0.1.1-rc.2 → 0.1.2-rc.1** changelog 的核对。

既有产品立场（不变）：对**跨会话、必须由人处理的工作**做自动草稿加一次 Pin/Keep 确认。不是经典待办应用。仅 agent（智能体）的未完成工作仍留在恢复、`todo/write`、目标、计划审阅和会话列表中。

-----

<a id="verdict"></a>
## 结论

**0.1.2-rc.1 是否补上了下班钉选 / 必须由人处理折叠所要的钩子？**

**否。**

对插件侧日志 API 作宽厚解读，最多也只是**实现机制上的部分**，不是所要钩子：

- Claude Code 的 `SessionEnd`、`FileChanged`、`Notification`、`TaskCompleted` 与 `TeammateIdle` 仍在 23 个不支持事件中。证据：[`packages/hooks/hooks-claude-code/src/config.ts`](../packages/hooks/hooks-claude-code/src/config.ts) 仍只枚举 `SessionStart`、`UserPromptSubmit`、`PreToolUse`、`PostToolUse`、`Stop`、`SubagentStart` 与 `SubagentStop`。该列表在 `dsh-v0.1.1-rc.2`、`dsh-v0.1.2-alpha.1`、`dsh-v0.1.2-rc.1` 与 `dsh-v0.1.3-alpha.1` 上相同。
- 已知限制仍把这 23 个事件标为在配置组解析前忽略：[hooks-claude-code 已知限制](../packages/hooks/hooks-claude-code/README.zh.md#known-limitations-and-deferred-work)。
- Codex 仍只映射五个事件（`PreToolUse`、`PostToolUse`、`SessionStart`、`UserPromptSubmit`、`Stop`），见 [`packages/hooks/hooks-codex/src/config.ts`](../packages/hooks/hooks-codex/src/config.ts)。
- rc.1 上的 `KNOWN_SESSION_EVENT_TYPES` 与 alpha.1 相同。没有 `session/end`、`pin/*`、`notification/*` 或文件监视会话事件。自 0.1.1-rc.2 起新增的 SessionEventMap 成员只有 `model/selection`、`session-log-deepseek/delivery-accepted` 与 `subagent/model-selection-policy`——都不是下班摘要。
- `packages/client/ui-trajectory` 中的 `SessionEndState` 是 Trajectory 压缩（compaction）Chat 节点，不是 Claude Code `SessionEnd` 钩子，也不是桌面摘要。

不要把 RC 发布说明里的「空闲」条目当成用户离开检测。WebSocket 心跳保持空闲**连接**存活。`agent/status` 的 idle 仍表示每个轮次之后「没有 driver 仍在调度」。

-----

<a id="wanted-hooks"></a>
## 所要钩子

| 所要 | 在 `dsh-v0.1.2-rc.1` 上？ | 证据 |
|---|---|---|
| Claude Code / 原生 `SessionEnd` | **否** | [hooks-claude-code README](../packages/hooks/hooks-claude-code/README.zh.md#known-limitations-and-deferred-work) 的不支持列表；[`config.ts`](../packages/hooks/hooks-claude-code/src/config.ts) 中的 `CLAUDE_EVENTS`；该标签下 `packages/core/session/src/known-event-types.ts` 没有 `session/end`。`session/end-seed` 仍是 fork/恢复日志标记。 |
| 日历 / 下班定时器 | **否** | Schedule 仍是会话内的 `after` / `at` / `every_seconds`（≥ 300s），没有 Cron，没有邮件/推送。冷会话会一直过期直到恢复。rc.1 的标题区目录是这些同一批记录的 UI，不是桌面时钟。 |
| 面向下班的空闲改进 | **否** | `agent/status` 与 `whenIdle()` / `runMaintenance()` 在 RC 之前就已存在。心跳与连接状态 UI 属于传输层，不是「人已下班」的去抖。 |
| 供宿主折叠的跨会话 `sessionQuery` | **本来就有；钉选工作未变** | 宿主 `listSessions()` / `filterSessions()` 仍无 cwd 锁（[`packages/session-query/session-query/src/index.ts`](../packages/session-query/session-query/src/index.ts)）。模型工具在 [`workspace-access.ts`](../packages/session-query/tool-session-query/src/workspace-access.ts) 仍按调用方精确 `cwd` 过滤。`observeSession()` 在 0.1.2-alpha.1 就已存在。 |
| 操作系统 / 产品 `Notification` 钩子 | **否** | Claude Code `Notification` 仍被跳过。ACP（Agent Client Protocol）/SDK 的「notification」载荷是协议更新，不是桌面推送。 |
| `FileChanged` | **否** | CC 桥仍不支持。[`dsh-fs`](../packages/fs/fs/src/index.ts) 仍只有按次调用的 `fs/write-intent`、`fs/edit-intent`、`fs/observed`——没有监视原语。 |
| `TaskCompleted` | **否** | 仍不支持。任务完成仍只唤醒**本会话**空闲 owner；不是跨会话钉选。 |
| 对新钉选卡片有用的 Cordis 事件 | **没有新的摘要事件** | 与钉选相关的 `interface Events` 名称在 alpha.1 与 rc.1 之间相同。`tools/code-dispatch-log` 在 alpha.1 上已是 `tools/ptc-dispatch-log`（更名发生在 0.1.1-rc.2 → rc.1 窗口），无助于钉选。 |

-----

<a id="what-012-rc1-actually-changed"></a>
## 0.1.2-rc.1 实际改了什么

这些是插件相邻的差异。其中没有一项是钉选钩子。

### 日志读取 API（插件在 rc.1 上必须适配）

在 `dsh-v0.1.2-rc.1` 上，`Session.events` 已不存在（在 `dsh-v0.1.2-alpha.4` 删除）。原先遍历完整日志的插件现在调用 `eventAt(seq)`、`snapshotEvents(from, toExclusive)`、`ownEvents()` 与 `seq`。见该标签下的 `packages/core/session/src/index.ts`（本 fork 仍有 `get events()`，因为它是 `0.1.2-alpha.1`）。针对本检出写成的 `/pin` 折叠在改用这些 API 之前，无法在 rc.1 上通过类型检查。

### 恢复 `ignorable`（树外持久附加项）

`0.1.2-alpha.1`（本 fork / 头脑风暴）去掉了 `SessionEvent.ignorable`。`dsh-v0.1.2-alpha.2` 恢复了它；rc.1 仍保留（该标签下的 `packages/core/session/src/types.ts`）。未知事件类型会拒绝重建，除非信封带有 `ignorable: true`。这是**树外**插件事件的兼容舱口，不是一等钉选存储：不认识该类型的构建会跳过该事件；树内 required-on-read 的 `pin/*` 成员仍会加入生成的 `KNOWN_SESSION_EVENT_TYPES`。

### 不是下班，不要误读

| rc.1 条目 | 为什么不是钉选 |
|---|---|
| 父级与可持续子级之间的 `send_message` | subagent 后续消息，取代单向 `report`。该工具包在 alpha.1 就已存在。 |
| 会话标题区的 Schedule 目录 | 渲染既有的会话内 `schedule/change` 记录。 |
| WebSocket 心跳 | 保持空闲 **socket** 存活（`packages/api/gateway`）。与「人已回家」相反。 |
| 连接状态 / 重试 UI | 传输世代（`connection/reset`），不是桌面空闲。 |
| Trajectory 的 `SessionEndState` | Trajectory 中的压缩节点，不是会话拆除。 |

### 更晚的标签，不是本次 RC

`dsh-v0.1.3-alpha.1`（2026-09-04 发布）**不是** 0.1.2-rc.1。它把会话格式升到 v2（`SESSION_FORMAT_VERSION = 2`），用 `assistant/attempt` 替换 `assistant/chunk`，并使持久化由生命周期持有的 `SessionHandle` 拥有，且 `agentLoop.create()` 改为异步。面向 rc.1 的插件不得假定这些约定。那会是另一次升级税。

-----

<a id="plugin-without-core"></a>
## 不改核心的插件

**可以**，缺失的下班*产品*仍可做成 Cordis 插件，而不编辑 `agent-loop`。头脑风暴时就已经如此。rc.1 没有改合法扩展点；它只改了插件如何读取会话日志。

原生钩子就是挂在已文档化扩展点上的普通插件（[架构](architecture.zh.md)，[扩展 cookbook](cookbook/extension-cookbook.zh.md#a-hook-plugin-permission-gate-example)）。`packages/hooks` 组仍是 Claude Code / Codex 兼容适配器，不是产品钩子 SDK。

### 插件今天可以监听什么

钉选插件可在不改循环的情况下使用的实时 Cordis 事件（本树中的声明；这些名称在 rc.1 上未变）：

| 事件 | 模式 | 钉选用法 |
|---|---|---|
| `agent/status` | emit | 最接近的安静信号；在插件 `Config` 中去抖 |
| `agent/session-start` | emit | 仅恢复/启动 |
| `agent/turn-stopping` | serial | Stop 钩子极性（引导再走一步）——对下班方向相反 |
| `agent/disposed`、`session/disposed` | emit | 进程丢掉了存活 agent，不是 18:00 |
| `session/event` | emit | 观察 `todo/write`、`goal/change`、`schedule/change`、`plan/mode`、审批、提问 |
| `session/created`、`session/flush` | emit / parallel | 启动 / 持久化检查点 |
| `tools/pre-execute`、`tools/post-execute`、`tools/result` | waterfall（瀑布式事件） / emit | 高流量；不是 FileChanged |
| `fs/write-intent`、`fs/edit-intent`、`fs/observed` | waterfall / emit | 按次写入，没有监视 |
| `approval/request`、`user-questions/request` | waterfall | 必须由人处理的路径已经在阻塞 |
| `subagent/start`、`subagent/end` | emit | 子级生命周期 |
| `commands/change` | emit | 注册表变动，不是摘要 |

插件可调用的宿主服务（宿主 API 无 cwd 锁）：

- `ctx.sessionQuery.listSessions()` / `filterSessions()` / `observeSession()` / `readSession()` / `readTitleSnapshots()`
- `ctx.commands` 用于注册 `/pin`
- `ctx.userQuestions.ask` 做 Pin / Keep / 关闭（plan-review 的 `intent` 已存在）
- `agent.whenIdle()` 与 `agent.runMaintenance(task)` 在公开状态保持 `idle` 时起草
- `ctx.webhookRuntime` 外加**外部**时钟，若有人坚持要 18:00

生成的[事件生产者-消费者矩阵](event-producer-consumer.zh.md)是**本次**检出的穷尽派发器/监听器列表。它在 rc.1 上不会多出 SessionEnd 行。

### 插件可不改核心而持久化什么

- harness 已经存储的仅日志事实，在 `/pin` 时折叠。
- rc.1 上的树外 `ignorable` 会话事件（不认识该类型的读取方会跳过）。
- 通过 `ConversationNodeDefinition` 做的 Web Chat 卡片是**客户端插件**，不是 `agent-loop` 变更。它仍是产品 UI，不在本任务的实现范围内。

面向模型的 `session_search` 仍受 cwd 锁定。不要用已交付工具让项目 A 中的模型去搜索项目 B。

-----

<a id="still-requires-core"></a>
## 仍需核心

这些仍需要新 seam、桥映射，或循环/日志词汇变更——0.1.2-rc.1 一项都没交付：

| 缺口 | 为什么插件无法诚实伪造 |
|---|---|
| `SessionEnd` / `session/end` | 桥会跳过；`session/disposed` 是进程拆除。补上 CC 映射属于钩子桥变更。 |
| 日历 Cron / 桌面范围 18:00 | Schedule 没有日历语言，也不会离开其会话。 |
| 操作系统推送 / Claude Code `Notification` | 没有 harness 通知通道。Webhook 会从外部时钟创建**新的**根会话。 |
| `FileChanged` 监视 | 没有 `fs.watch`。指令文件在成功读/写/编辑时重载，而不是在外部编辑时。 |
| 作为桌面事件的 `TaskCompleted` | 任务只唤醒所属会话。 |
| 跨会话 todo / 目标 / Schedule 联合 | 这些记录是**每个会话** last-write-wins。 |
| 树内 required-on-read 的 `pin/*` | 加入 `SessionEventMap` 会更新 `KNOWN_SESSION_EVENT_TYPES` 与持久化目录。 |
| 在 18:00 唤醒**冷**主会话 | 持久化会保存过期的 Schedule 记录；没有东西会呼叫用户。 |

-----

<a id="product-lean"></a>
## 产品立场

Product God 要对**跨会话**、必须由人处理的工作做自动草稿 + Pin/Keep，而不是再做一个待办列表。

rc.1 没有推动这根针。有区分度的工作仍是折叠人忘掉的**其他 / 冷**会话。宿主 `sessionQuery` 已经能列出它们。确认 UX 已存在于 `userQuestions.ask`（以及计划模式审阅）。当前打开会话里的 todo 仍是错误的存储。

推荐实验，仍可选，仍未排期：一个 patch 层插件，注册 `/pin`、折叠宿主查询，并询问 Pin/Keep。空闲去抖与 Schedule `at` 以后可触发同一折叠。不要为此加 `SessionEnd`。不要等待并不存在的 0.1.2-rc.1「钩子」。

-----

<a id="dev-note"></a>
## Dev Note

<details>
<summary>工作上下文 — 点击展开</summary>

通过拉取 `upstream` 标签 `dsh-v0.1.1-rc.2`、`dsh-v0.1.2-alpha.1` 至 `dsh-v0.1.2-alpha.5`、`dsh-v0.1.2-rc.1` 与 `dsh-v0.1.3-alpha.1` 完成核对。本 fork 的 `master` 是 `2c723c534a`（`0.1.2-alpha.1` 加五个本地提交）；与 rc.1 的 merge-base 是 `cd5ef81481`（alpha.1）；`git rev-list --count HEAD..dsh-v0.1.2-rc.1` 为 656。先前清单：[PR #3 头脑风暴](https://github.com/aa2246740/deepseek-harness/blob/cursor/eod-pin-brainstorm-a0fb/docs/eod-pin-brainstorm.md)。「不要把 rc.1 当成已加入钉选钩子」的决策所有者：[Agent Note](../.agents/notes/proposed/feature/2026-09-05-eod-pin-hooks-0.1.2-rc1.zh.md)。

</details>
