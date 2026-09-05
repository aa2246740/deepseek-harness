# Agent Note: 0.1.2-rc.1 没有加入下班钉选钩子

Status: proposed

[English](2026-09-05-eod-pin-hooks-0.1.2-rc1.md) | 中文

## 问题

Product God 询问 DeepSeek Harness 0.1.2-rc.1 是否加入了 `SessionEnd`、日历下班定时器、把空闲当作离开、操作系统通知、`FileChanged`、`TaskCompleted`，或其他能解锁下班钉选 / 必须由人处理折叠的 Cordis 事件。若把该 RC 当成已有这些钩子，会把插件或核心改动带上错误路径。先前清单在 [PR（Pull Request） #3](https://github.com/aa2246740/deepseek-harness/pull/3)，是对本 fork 的 `0.1.2-alpha.1` 检出写成的。上游中间标签 `dsh-v0.1.2-alpha.2` 至 `alpha.5` 也没有加入所要钩子。

## 提案

不要把官方标签 `dsh-v0.1.2-rc.1` 当成已加入那些钩子。证据页是 [0.1.2-rc.1 钩子差异](../../../../docs/eod-pin-hooks-0.1.2-rc1.zh.md)。

保持既有产品立场：可选 Cordis 插件（`/pin` + 宿主 `ctx.sessionQuery` + `ctx.userQuestions.ask`），不是原生卡片，不是 Claude Code `SessionEnd`，也不是待办应用。面向 rc.1 的插件必须用 `eventAt` / `snapshotEvents` 读日志，而不是 `Session.events`。树外 `ignorable` 事件在 alpha.2 恢复并保留到 rc.1，但不是一等钉选存储。

本 Agent Note 不实现该插件。它不取代 [Task Surface](2026-08-04-task-surface.zh.md)（单会话结构化表单）。

## 考虑过的替代方案

- **把 rc.1 的「空闲」/ 心跳 / Schedule 标题 / `send_message` 当成钉选：** 否决。那些是连接保活、会话内 Schedule UI，以及 subagent 后续消息。
- **等到 `SessionEnd` 再实验：** 否决。Claude Code 桥在 rc.1 与 `dsh-v0.1.3-alpha.1` 上仍跳过它。
- **假定本 fork 就是 0.1.2-rc.1：** 否决。本次检出是 `0.1.2-alpha.1`，落后该标签 656 个提交。
- **假定所要钩子在 `dsh-v0.1.2-alpha.2`–`alpha.5` 出现并在 rc.1 前被删：** 否决。这些标签保持相同的 `CLAUDE_EVENTS` 列表与 `KNOWN_SESSION_EVENT_TYPES` 成员。

## 验收标准

- 清单页把 `dsh-v0.1.2-rc.1` 标为确切标签，对所要钩子给出 **否**，并引用路径。
- 本变更不交付 `SessionEnd` 映射、`session/end` 事件或钉选 Chat 卡片。
- 若以后做钉选插件，仍消费 `ctx.commands`、宿主 `ctx.sessionQuery` 与 `ctx.userQuestions`，而不编辑 `agent-loop`。

## 风险

- 因 RC changelog 中无关的「空闲」措辞再次争论 `SessionEnd`。
- 针对本 fork 的 `Session.events` getter 编写 `/pin` 折叠，却发现 rc.1 上并不存在。
- 把 `dsh-v0.1.3-alpha.1` 的 `SessionHandle` / 格式 v2 与本次 RC 混淆。
