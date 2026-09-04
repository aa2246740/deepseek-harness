# 下班钉选卡片 — 钩子清单与可行性

[English](eod-pin-brainstorm.md) | 中文

仅供讨论与可行性评估。本页不是产品约定，除现有钩子、事件与 API 的清单外，不描述已交付行为。

## 摘要

DeepSeek Harness 已经把当天工作记在各会话的事件日志、标题、todos、goals、plan mode 审阅、审批、提问、jobs 与提醒里。它没有 `SessionEnd` 钩子、没有日历意义上的下班定时器、没有跨会话 todo 列表，也没有操作系统通知通道。原生「钉选今天的工作」卡片对纯 agent（智能体）编程大多是重复已有界面；只有作为可选宿主插件、为必须行动的人折叠**其他会话**时，才值得存在。

## 目录

- [钩子清单](#hooks-inventory)
- [可行性](#feasibility)
- [方案头脑风暴](#brainstorm)
- [何时有用](#when-useful)
- [结论](#verdict)
- [开发者备注](#dev-note)

-----

<a id="hooks-inventory"></a>
## 钩子清单

本清单只引用本仓库里真实存在的 API。Claude Code 或 Codex 桥接跳过的事件，这里不会虚构。

### 这里的「钩子」指什么

**原生钩子**就是挂在已文档化扩展点上的普通 Cordis 插件。没有单独的 native-hooks 包。该规则记录在[拦截扩展点 Agent Note](../.agents/notes/implemented/feature/2026-06-30-interception-extension-points.zh.md)，并在[扩展 cookbook](cookbook/extension-cookbook.zh.md#a-hook-plugin-permission-gate-example)中重述。

`packages/hooks` 组是现有 Claude Code 与 Codex `hooks.json` 命令钩子的**兼容适配器**。定制产品行为应挂在桥接已经编程的同一批 Cordis 事件上，而不是新的 `hooks.json` 方言。见[hooks 组地图](../packages/hooks/README.zh.md)、[hook-protocol](../packages/hooks/hook-protocol/README.zh.md)、[hooks-claude-code](../packages/hooks/hooks-claude-code/README.zh.md) 与 [hooks-codex](../packages/hooks/hooks-codex/README.zh.md)。

新行为应挂到哪里，见 [architecture.md](architecture.zh.md#where-new-behavior-goes)。生成的[事件生产者—消费者矩阵](event-producer-consumer.zh.md)列出每个 harness 事件的分发者与监听器。

### 实际会触发的 Claude Code 与 Codex 钩子点

两条桥接只把一小部分命令钩子映射到 harness 事件。共享的持久记录是仅日志的 `hook/invoked` 与 `hook/result`，必须落在打开的轮次内，声明于 [`packages/hooks/hook-protocol/src/events.ts`](../packages/hooks/hook-protocol/src/events.ts)。`SessionStart` 在第 1 轮之前运行，**不会**追加 `hook/*`；它改为注入上下文。

| 外部钩子 | Harness 扩展点 | 今天能做什么 | 与钉选卡片的关系 |
|---|---|---|---|
| `SessionStart` | `agent/session-start` emit | 附加上下文；脱离运行，可能错过第一次请求 | 会话启动／恢复，不是下班 |
| `UserPromptSubmit` | `agent/pre-step` waterfall | 拦截提示词或附加上下文 | 用户打字时按需钉选，不是空闲下班 |
| `PreToolUse` | `tools/pre-execute` waterfall | 拒绝（Claude Code 可以 `ask`） | 工具门禁，不是摘要 |
| `PostToolUse` | `tools/post-execute` waterfall | 用反馈拦截结果，或附加上下文 | 可以观察写入，但每次工具都会触发 |
| `Stop` | `agent/turn-stopping` serial | 通过 `steer()` **再强制一步**模型步骤 | 与「一天结束」相反 |
| `SubagentStart` / `SubagentStop` | `subagent/start`、`subagent/end` | 仅 Claude Code；stop 只观察 | 子生命周期，不是桌面摘要 |

`{"continue": false}` 会折入 `hook/result`，但**没有运行级硬停止**。没有 `SessionEnd` 映射。Claude Code 的 `FileChanged`、`Notification`、`TeammateIdle`、`TaskCompleted`、`PreCompact`、`PostCompact` 与 `SessionEnd` 在解析时被列为不支持并忽略，见 [hooks-claude-code 已知限制](../packages/hooks/hooks-claude-code/README.zh.md#known-limitations-and-deferred-work)。Codex 同样丢弃 `PermissionRequest`、`PreCompact`、`PostCompact`、`SubagentStart` 与 `SubagentStop`（[hooks-codex](../packages/hooks/hooks-codex/README.zh.md#known-limitations-and-deferred-work)）。

### Agent 与会话生命周期（真正的原生点）

声明于 [`packages/core/agent/src/runtime-types.ts`](../packages/core/agent/src/runtime-types.ts) 与 [`packages/core/session/src/index.ts`](../packages/core/session/src/index.ts)。轮次流程见 [architecture.md](architecture.zh.md#turn-flow) 与 [agent-lifecycle.md](agent-lifecycle.zh.md)。

| 事件或 API | 模式 | 何时运行 | 与钉选卡片的关系 |
|---|---|---|---|
| `agent/created` | emit | 已发布活的 agent | 太早 |
| `agent/session-start` | emit | 第 1 轮前一次；`source` 为 `startup` \| `resume` \| `clear` \| `compact` | 恢复是真实的；不是下班 |
| `agent/status` | emit | `idle` ⇄ `running`；idle 表示没有仍在调度或活动的 driver | 最接近「会话安静」的信号；**每一轮**结束后都会触发 |
| `agent.whenIdle()` | 方法 | 当前整段 agent 活动达到静默 | 与 idle 相同；[schedule](../packages/schedule/schedule/README.zh.md) 在用 |
| `agent.runMaintenance(task)` | 方法 | 从真正 idle 跑一项非轮次任务；对外状态保持 `idle` | 适合起草摘要且不打开轮次 |
| `agent/pre-step` | waterfall | 每一个拟议步骤 | 可以注入上下文；不是下班定时器 |
| `agent/turn-stopping` | serial | 模型已无待响应；监听器可 `steer()` 继续 | Stop 钩子语义，不是钉选 |
| `agent/disposed` | emit | 静默后离开 registry，会话分离之前 | 进程拆除，不是下班 |
| `session/created` | emit | 会话进入活存储 | 启动 |
| `session/event` | emit | 每一次追加 | 观察日志、todos、工具、文档写入的通用点 |
| `session/flush` | parallel | 持久化检查点 | 持久化，不是 UX |
| `session/disposed` | emit | 活存储条目离开 | 关标签／进程，不是日历下班 |

`SessionStartSource` 为 `'startup' | 'resume' | 'clear' | 'compact'`（[`runtime-types.ts`](../packages/core/agent/src/runtime-types.ts)）。恢复是在 `ctx.sessionPersistence.load` 之后调用 `ctx.agents.resume({ resumeSessionId })`（[session-persistence](../packages/session/session-persistence/README.zh.md)、[agent registry](../packages/core/agent/src/index.ts)）。

**没有** `session/end` 会话事件，也**没有** harness 的 `SessionEnd` 钩子。`session/end-seed` 是日志上的 fork／恢复种子边界标记，不是进程拆除或日历下班。适配器里的 LLM idle 看门狗是流读取超时，不是检测用户离开。

### 摘要可以折叠的持久会话事实

会话日志是事实来源（[subsystems/session.md](subsystems/session.zh.md)、[持久化目录](persistence-catalog.zh.md)）。每会话状态在回放时后写覆盖；它不是跨项目看板。

| 事实 | 事件／API | 范围 | 说明 |
|---|---|---|---|
| 轮次、消息、工具调用 | `turn/*`、`user/message`、`assistant/message`、`tool/call`、`tool/result` | 一个会话 | 当天工作的完整 transcript |
| 标题 | `session/title`，经 [`dsh-session-title`](../packages/session/session-title/README.zh.md) | 一个会话 | 列表行名称；用户重命名会钉住，不再自动刷新 |
| Todos | `todo/write`，经 [`dsh-tool-todo`](../packages/todo/tool-todo/README.zh.md) | 一个 agent 会话 | 整表替换；**不在** agent 之间共享；UI 投影在**下一轮开始时清空** |
| Plan mode | `plan/mode`，经 [`dsh-plan-mode`](../packages/plan/plan-mode/README.zh.md) | 一个 agent | 软引导；`exit_plan_mode` 已通过 [user-questions](../packages/interaction/user-questions/README.zh.md) 给出 Approve／Keep planning |
| Goals | `goal/change`，经 [`dsh-goal`](subsystems/goal.zh.md) | 同一会话 | 阶段为 `active` \| `paused` \| `blocked` \| `complete`；[goal-round-driver](../packages/goal/goal-round-driver/README.zh.md) 在 idle 且已武装时自动续跑 |
| Schedule | `schedule/change`，经 [`dsh-schedule`](subsystems/schedule.zh.md) | 同一会话 | `after`／`at`／`every_seconds`（≥ 300s）；**没有日历／Cron**；**没有邮件／推送**；冷会话逾期直到该会话被恢复 |
| 审批 | `approval/asked`、`approval/decided` | 打开的轮次 | 人必须行动，已经在阻塞 |
| 命令 | `command/*` | 一个会话 | `/plan`、`/goal`、`/compact` 不创建模型消息 |
| Compaction | `compaction/*` | 一个会话 | 历史摘要，不是每日钉选 |
| 种子边界 | `session/end-seed` | 一个会话 | fork／恢复标记；**不是** SessionEnd 或日历下班 |
| Workflow 运行 | `tool-workflow/*`、`workflow/*` 实时事件 | 一个会话／引擎 | 前台编排 |
| 实验性团队看板 | `team/task` | 一个会话 | 私有实验包，不是正式发布面 |

文件系统写入不是「文档钩子」。[`dsh-fs`](../packages/fs/fs/README.zh.md) 没有 watch 原语。`fs/write-intent`、`fs/edit-intent` 与 `fs/observed` 是每次调用的门禁（[fs-observation-policy](../packages/fs/fs-observation-policy/README.zh.md)）。[`dsh-agent-instructions`](../packages/context/agent-instructions/README.zh.md) 在成功的 read／write／edit 或恢复基线时加载 `AGENTS.md`／`CLAUDE.md` — **没有文件监视器**。[`dsh-skill-filesystem`](../packages/skill/skill-filesystem/README.zh.md) 只监视 skill 根目录。

### 跨会话 API（「并行项目」的唯一路径）

Todos、goals、plan 与 schedule 都是**会话本地**的。宿主仍然能看见许多会话：

| API | 包 | 返回什么 | 授权 |
|---|---|---|---|
| `ctx.sessionPersistence.list()` | [session-persistence](../packages/session/session-persistence/README.zh.md) | 已存储的 header | 宿主 |
| `ctx.sessionQuery.listSessions()`／`filterSessions()` | [session-query](../packages/session-query/session-query/README.zh.md) | 以活会话优先的语料；可按 cwd、创建时间、父会话过滤 | 宿主；无 cwd 锁 |
| `ctx.sessionQuery.searchSessions()` | [session-query-sqlite](../packages/session-query/session-query-sqlite/README.zh.md) | 排序后的 FTS | 宿主；可选索引 |
| `sessionController.list()` | [session-controller](../packages/api/session-controller/README.zh.md) | 带标题、`running`、`lastPromptAt` 的摘要；不激活 agent | 宿主／Web 列表 |
| `session_search` 及同组工具 | [tool-session-query](../packages/session-query/tool-session-query/README.zh.md) | 面向模型的搜索 | **精确 `cwd` 匹配**；未挂入已交付的宿主组合 |
| `ctx.sessionReferenceResolver.listCandidates()` | [session-reference](../packages/context/session-reference/README.zh.md) | 按 cwd 亲和排序的其他会话，供 `@` 提及 | 宿主提及 UX |

在项目 A 里运行的模型不能用已交付的 session-query **工具**去读项目 B。宿主插件或斜杠命令**可以**。

### 产品上已有的人机门禁与时钟 seam

| Seam | 角色 | 已经是「钉选」吗？ |
|---|---|---|
| [`dsh-user-questions`](../packages/interaction/user-questions/README.zh.md) + [`ask_user_question`](../packages/interaction/tool-ask-user/README.zh.md) | 暂停直到人回答 | 是，阻塞 |
| [`dsh-user-approval`](../packages/interaction/user-approval/README.zh.md) | 对一次工具调用允许／拒绝 | 是，阻塞 |
| Plan mode 的 `exit_plan_mode` | Approve／Keep planning | 是，结构化确认 |
| [`dsh-commands`](../packages/interaction/commands/README.zh.md) | 不经模型轮次的 `/name` | 自然的 `/pin` 挂载点 |
| [`dsh-schedule`](../packages/schedule/schedule/README.zh.md) | 稍后作为普通用户消息的会话本地提醒 | 最接近的时钟；不是整张桌子 |
| [`dsh-time-context`](../packages/context/time-context/README.zh.md) | 每步给模型的时钟文本 | 不是定时器 |
| [`dsh-tool-jobs`](../packages/jobs/tool-jobs/README.zh.md) | 任务完成时唤醒空闲 owner | 未完成工作，仍在会话内 |
| [`dsh-webhook`](../packages/webhook/webhook/README.zh.md) | 外部投递 → 新的根会话 | 需要**外部**时钟 |
| 提案 [Task Surface](../.agents/notes/proposed/feature/2026-08-04-task-surface.zh.md) | 声明式单会话表单；`show_task_surface` 结束该轮 | 若落地，会与「Pin／Keep」UI 重叠；不是下班摘要 |

为新的持久事实做 Web Chat 卡片要注册 `ConversationNodeDefinition`（[conversation 子系统](subsystems/conversation.zh.md)）。那是产品 UI，超出本页范围。

-----

<a id="feasibility"></a>
## 可行性

可行性假定要做「当天工作 + 并行项目未完成工作」的摘要，且 Product God 倾向自动草稿加一次 Pin／Keep 确认。评级：**easy**（挂在现有事件／API 上的插件）、**needs plugin**（新包，不改 loop）、**needs core**（新生命周期事件、改 loop，或新的跨会话存储）。

| 候选输入 | 今天存在吗？ | 插件会怎么用 | 评级 | 缺口 |
|---|---|---|---|---|
| 会话结束／进程退出 | `session/disposed`、`agent/disposed` | 观察拆除 | easy | 表示「本进程丢掉了活 agent」，不是 18:00 |
| 一轮之后的 idle | `agent/status` idle、`whenIdle`、`runMaintenance` | 在状态保持 idle 时起草 | easy | 每一轮都会触发；需要 harness 并未作为下班提供的防抖／时钟 |
| 计划完成 | `exit_plan_mode` + `plan/mode` | 已经是审阅 | easy | 只限该会话，且仅在规划中 |
| 工具结果／文档写入 | `session/event` 的 `tool/result`、`fs/write-intent` | 统计写入 | easy | 量大；没有 `FileChanged` 监视器；指令文件没有 watch |
| Claude Code `Stop` | `agent/turn-stopping` | 会**继续**这次运行 | easy | 对下班极性反了 |
| Claude Code `SessionEnd` | **无** | — | needs core | 桥接明确跳过；不要为钉选卡片去加 |
| 日历 18:00 | Schedule 的 `at`，带 `{ date, time, time_zone }` | 在**那个**会话里提醒 | needs plugin | 会话本地；冷会话要等到恢复；无操作系统通知 |
| 每日重复 | Schedule 的 `every_seconds` ≥ 300 | 从创建起定频，不是墙上时钟 18:00 | needs plugin | 不对齐日历；仍是会话本地 |
| 跨项目列表 | `sessionQuery.listSessions`／`filterSessions` 的 created-at | 宿主折叠标题、最近 `todo/write`、goal 阶段 | easy | 模型工具不能跨 cwd；宿主／命令可以 |
| 共享未完成工作存储 | **无** | 新的 `SessionEventMap` 或旁路数据库 | needs core | Todos／goals／schedule 不会联合 |
| 整桌推送 | **无** | Webhook + 外部 cron，或操作系统通知 | needs plugin + 外部时钟 | Webhook 会创建**新**会话；schedule 从不离开其会话 |

加入 `SessionEnd`、进程级每日定时器或跨会话 todo 看板会是核心（或新 seam）变更。交付一个读取 harness 已存储日志的 `/pin` 命令，并不需要这些。

-----

<a id="brainstorm"></a>
## 方案头脑风暴

按对**真实存在的钩子与 API** 的贴合度排序，而不是外观。下列都不是已排期工作。

### 1. 按需 `/pin` 命令 + 宿主查询折叠 + Pin／Keep 提问 — 最贴合

插件在 [`ctx.commands`](../packages/interaction/commands/README.zh.md) 上注册 `/pin`。处理函数调用 `ctx.sessionQuery.listSessions`／`filterSessions`（今天的 created-at，可选 cwd 列表），加载各日志或投影，折叠标题、最近 `todo/write`、goal 阶段、plan-mode 标志与逾期 `schedule` 记录。然后用 Approve 风格选项（Pin／Keep／dismiss）调用 `ctx.userQuestions.ask`，复用 plan mode 已有确认路径。若需要持久 Keep，可在选定的「desk」会话上写仅日志事件，或做一个小投影单元 — 仍是插件，不是 loop 变更。

这符合 Product God 的自动草稿 + 一次确认，且**不必发明下班时刻**。用户（或稍后的 idle 触发）决定何时运行。跨项目可行，因为宿主查询没有 cwd 锁。

### 2. 在活会话里 idle 自动起草

监听 `agent/status` → `idle`，再 `runMaintenance` 做同样的折叠，`inject` 或 `followup` 一份草稿，然后 `ask`。easy 插件。Harness 的 idle 信号是**每一轮**，幼稚监听器会在每次回复后打扰。必须做配置防抖（按小时，不是按秒）；该防抖是插件 `Config` 字段，不是新的核心事件。

### 3. 在一个 home 会话里用 Schedule `at` 本地傍晚

使用现有 [Schedule overlay](user/guide/schedule.zh.md)：用浏览器时区的 `LocalAtInput` 在 18:00 调用 `schedule_create`。投递是该会话在 `whenIdle` 之后的普通 follow-up。easy overlay／插件。并行项目不可见，除非 home 会话的插件也读 `sessionQuery`。冷的 home 会话把逾期提醒存到恢复为止；不会到达用户手机。

### 4. 复用已交付的列表 + todos + goals + plan 审阅 — 已经交付

Web 会话列表已经显示标题、running 状态与上次提示时间（[session-controller list](../packages/api/session-controller/src/list.ts)）。Todos、goals 与 plan 审阅已经把会话内未完成工作摆出来。此方案是文档与 UX 强调，不是新卡片。对 **agent-only** 工作，这是回应 Cola 疑虑的最强答案。

### 5. 外部时钟 → webhook → 摘要会话

[`ctx.webhookRuntime`](../packages/webhook/webhook/README.zh.md) 可以从一次受信任投递创建新的根会话。18:00 的操作系统 cron 或日历应用是 DSH 并不拥有的时钟。新会话的提示词可以让模型做摘要；宿主侧折叠仍比指望模型搜索其他 cwd 更诚实（已交付的 `session_search` 会拒绝）。需要插件规则加外部调度器。

### 6. 把 `exit_plan_mode` 当作钉选

Plan mode 已经会为 Approve／Keep planning 停下。人必须在执行前批准计划时有用。作为「今天跨项目」无用，且 agent 不在 plan mode 时不会出现。

### 7. 在 `hooks.json` 里用 Claude Code `Stop`／期望中的 `SessionEnd`

`Stop` 会再强制一步。`SessionEnd` 未实现。把兼容桥接当产品钉选，会与它们文档化的限制冲突（[hook 桥接 Agent Note](../.agents/notes/implemented/feature/2026-06-30-hook-bridges.zh.md)）。为本想法拒绝。

### 8. 新的 `pin/*` 事件 + Chat 卡片 + 核心存储

扩展 `SessionEventMap`，加投影，注册 `ConversationNodeDefinition`。这是增加持久产品状态的架构合法路径（[architecture.md](architecture.zh.md#where-new-behavior-goes)）。它需要本任务不实现的核心邻近包与 Web UI。提案 [Task Surface](../.agents/notes/proposed/feature/2026-08-04-task-surface.zh.md) 已经覆盖单会话结构化确认；它不列出其他项目。除非插件实验（方案 1）证明有人机协作的 owner，否则不要开工。

-----

<a id="when-useful"></a>
## 何时有用

对默认编程循环，Cola 的疑虑是对的。Product God 的卡片只在更窄的切片里有用。

### 纯 agent 工作（规划、执行、恢复）— 大多无用

编程 agent 已经会：

- 用 `todo_write` 写一份站立计划（日志支撑；恢复时重建）；
- 进入 `/plan`，再用 `exit_plan_mode` 做人机门禁，或跳过规划直接跑；
- 通过 `ctx.goals` 与 idle round driver 继续同一会话目标；
- 在会话活着时因 job 完成与到期 schedule 记录而唤醒；
- 在 `ctx.agents.resume` 时重建整份日志。

未完成的 agent 工作**就在日志里**。下一会话不需要钉选才知道做什么。列出「仍 pending 的 todos」的钉选卡片重复 `todo/write`。列出「agent 写过的文件」重复 git 与 transcript。规划后立即执行的 agent 不会等早晨的钉选。

Todo **UI** 投影在下一轮清空（[todo README](../packages/todo/tool-todo/README.zh.md)）是站立计划寿命的选择，不是未完成工作消失的证明 — 最近一次 `todo/write` 仍在日志里，恢复时也在。

### 人必须行动的工作 — 已经在阻塞，额外钉选很弱

这些已经会停下 loop 或工具：

- `ask_user_question`／`user-questions/request`
- `approval/request`
- plan mode 审阅（`intent: { kind: 'plan-review' }`）
- `goal` 阶段 `blocked`

把它们再钉到第二张卡片上，等于重复一个未回答的问题。真正有用的剩余是**人忘掉、且当前不在屏幕上的工作**：另一会话里被 blocked 的 goal、**冷**会话上的逾期提醒（Schedule 不会在该会话外通知），或 agent 在不同 cwd 写下、要给人看的文档。

### 跨项目／人机协作 — 唯一有区分度的工作

本仓库里的「并行项目」是并行**会话**，通常有不同的 `SessionHeader.cwd`。没有共享 todo。实验性 agent-teams 在**一个会话内**共享看板，且排除在正式发布之外。

有区分度的工作是：在人选定的时刻，把**许多**会话 header 与最近的仅日志快照折成一份人可以 Keep 或 dismiss 的清单。用户碰巧打开的那个会话里的 `todo_write` 做不到这件事。今天宿主 `sessionQuery` **可以**做到。它不需要 `SessionEnd`。

### 全自动 vs 手动 vs 可选

| 触发 | 诚实对应 | 风险 |
|---|---|---|
| 18:00 全自动 | 不是 harness 钩子；一个会话里的 Schedule 或外部 cron | 错过冷会话；无推送 |
| idle 全自动 | `agent/status` idle | 除非防抖，每一轮都打扰 |
| 手动 `/pin` | `ctx.commands` | 符合「按需」；没有假时钟 |
| 自动草稿 + Pin／Keep | 命令或防抖 idle，然后 `userQuestions.ask` | Product God 倾向；仍是插件 |

建议用**手动或用户选择的自动草稿**，不要新的 `SessionEnd`。

-----

<a id="verdict"></a>
## 结论

**不要在核心里做原生钉选卡片，也不要为此增加 `SessionEnd` 钩子。**

不要在 `agent-loop` 里交付任何东西。Loop 已经暴露 idle、turn-stopping、恢复与会话日志。钉选产品应是这些点的 **Consumer**，不是新的 driver。

**若以后值得实验，做成可选插件**（命令 + `ctx.sessionQuery` 折叠 + `ctx.userQuestions` 确认），从 patch 或 bundle 挂载，而不是 Claude Code `hooks.json` 功能。稍后可以用 idle 或 Schedule 触发同一套折叠。只有在该插件有人机协作 owner、并证明会话列表 + todos + goals + plan 审阅不够之后，才有理由做持久 `pin/*` 事件与 Chat 卡片。

**不要**为了给这个想法让路而去建跨会话 todo 存储、操作系统通知，或在 Schedule 里做日历 Cron。那些会是带自己 Consumer 的新 seam，不是一张钉选卡片。

Cola 的默认：纯 agent 编程请用恢复、todos、goals 与会话列表。Product God 的确认 UX 已经作为 plan mode 审阅与 `ask_user_question` 存在；复用它们，而不是新的卡片类型。

-----

<a id="dev-note"></a>
## 开发者备注

<details>
<summary>维护者工作上下文 — 点击展开</summary>

本页是证据清单与排序后的方案。现行决定（不要原生钉选、不要 `SessionEnd`、仅当以后实验有人机协作 owner 时才做插件）记在 [EOD 钉选卡片 Agent Note](../.agents/notes/proposed/feature/2026-09-04-eod-pin-card.zh.md)。与 UI 重叠、但不是下班的在途提案：[Task Surface](../.agents/notes/proposed/feature/2026-08-04-task-surface.zh.md)（结构化单会话表单）与[交互式旁路会话](../.agents/notes/proposed/feature/2026-07-08-interactive-side-sessions.zh.md)（fork + 合并回写）。两者都不能替代跨会话摘要。若实现方案 1，应在一份取代该延后决定的新 Note 中写明插件包名与（若有）`SessionEventMap` 成员。

</details>
