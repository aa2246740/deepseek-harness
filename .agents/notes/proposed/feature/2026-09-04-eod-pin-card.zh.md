# Agent Note: 延后原生下班钉选卡片

Status: proposed

[English](2026-09-04-eod-pin-card.md) | 中文

## 问题

有一个产品想法希望用户在临近下班或按需时钉选一张卡片，列出当天工作以及并行项目中的未完成工作。Agent 会话已经写入大量日志，最容易想到的实现是增加 `SessionEnd` 钩子、原生 Chat 卡片，或跨会话 todo 看板。本仓库里的编程 agent 已经会从该日志规划、执行并恢复，因此第二块未完成工作看板可能只是重复 harness 已经交付的界面。

## 提案

不要增加原生钉选卡片、`SessionEnd` 钩子、`session/end` 会话事件，或 `SessionEventMap` 的 `pin/*` 成员。不要为这个想法改 `agent-loop`。真实存在的钩子、事件与 API 的证据清单，以及排序后的方案，见[下班钉选卡片头脑风暴](../../../../docs/eod-pin-brainstorm.zh.md)。

若以后值得实验，做成**可选插件**：在 `ctx.commands` 上注册 `/pin`，用宿主 `ctx.sessionQuery.listSessions()`／`filterSessions()` 折叠其他会话（无 cwd 锁），再用 `ctx.userQuestions.ask` 确认（Pin／Keep／dismiss），复用 plan mode 已有审阅路径。稍后可用 idle（`agent/status` → `idle` 加配置防抖）或在一个 home 会话里用 Schedule `at` 触发同一套折叠。本次变更不实现该插件。

纯 agent 编程继续使用恢复、`todo/write`、goals、plan mode 审阅与会话列表。钉选唯一能新增的有区分度工作，是折叠人忘掉的**冷会话或其他 cwd 会话**；宿主查询今天就能做这件事，不需要新的生命周期事件。

## 相关提案

本 Note 不取代 [Task Surface](2026-08-04-task-surface.zh.md)（单会话结构化表单）或[交互式旁路会话](2026-07-08-interactive-side-sessions.zh.md)（fork 加合并回写）。两者都不会在下班时列出其他项目。

## 曾考虑的替代方案

- **增加 Claude Code 的 `SessionEnd`（或把 `Stop` 当作下班）：**否决。Claude Code 与 Codex 桥接跳过 `SessionEnd`；`Stop` 映射到 `agent/turn-stopping` 并会**再 steer 一步**。原生钩子就是挂在现有扩展点上的普通插件（[拦截扩展点](../../implemented/feature/2026-06-30-interception-extension-points.zh.md)、[hook 桥接](../../implemented/feature/2026-06-30-hook-bridges.zh.md)）。
- **现在就交付原生 Chat 卡片与 `pin/*` 事件：**延后，直到插件实验有人机协作 owner，并证明会话列表、todos、goals 与 plan 审阅不够。那时持久产品状态再遵循[新行为应挂到哪里](../../../../docs/architecture.zh.md#where-new-behavior-goes)。
- **增加跨会话 todo 存储、操作系统通知，或在 Schedule 里做日历 Cron：**为本想法否决。Todos、goals 与 Schedule 都是会话本地的；Schedule 没有邮件／推送，也不会唤醒冷会话。那些会是新 seam，不是钉选卡片。
- **在每次 `agent/status` idle 上全自动：**否决作为默认触发。Idle 每一轮都会触发；防抖会是插件 `Config`，不是新的核心事件，除非用户选择加入，否则仍会打扰。

## 验收标准

- 本次变更不增加 `SessionEnd` 映射、不增加 `session/end` 事件、不增加 `pin/*` 会话事件，也不增加产品 Chat 卡片。
- 以后若做钉选插件，应消费 `ctx.commands`、宿主 `ctx.sessionQuery` 与 `ctx.userQuestions`，而不编辑 `agent-loop`。
- 纯 agent 的未完成工作继续使用恢复、最近一次 `todo/write`、goals、plan mode 审阅与会话列表，而不是第二块看板。

## 风险

- 因为 Claude Code 把该钩子叫做 `SessionEnd` 而再次争论要不要加它；清单页是引用代码的答案。
- 把「以后可选插件」当成已排期工作；它不是。在有人机协作 owner 之前什么都不交付。
- 宿主折叠许多会话日志可能很慢；任何插件都必须限制打开多少日志，并优先使用 header 以及最近的 `todo/write`／`goal/change`／`schedule/change` 快照。
