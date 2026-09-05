# EOD pin hooks vs 0.1.2-rc.1

English | [中文](eod-pin-hooks-0.1.2-rc1.zh.md)

Discussion and version-delta only. This page is not a product contract. It answers whether official DeepSeek Harness **0.1.2-rc.1** added the lifecycle hooks the prior EOD pin-card brainstorm asked for.

## Summary

**No.** Tag `dsh-v0.1.2-rc.1` (release name v0.1.2-rc.1) did not add `SessionEnd`, a calendar end-of-day timer, OS notifications, a `FileChanged` watcher, or `TaskCompleted`. The Claude Code bridge still ignores those events at parse time. A Cordis plugin can still implement auto-draft + Pin/Keep on APIs that already existed. This fork remains at `0.1.2-alpha.1` and is hundreds of commits behind that tag.

## Table of Contents

- [Scope](#scope)
- [Verdict](#verdict)
- [Wanted hooks](#wanted-hooks)
- [What 0.1.2-rc.1 actually changed](#what-012-rc1-actually-changed)
- [Plugin without core](#plugin-without-core)
- [Still requires core](#still-requires-core)
- [Product lean](#product-lean)
- [Dev Note](#dev-note)

-----

<a id="scope"></a>
## Scope

Compared tags on [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) (this fork has no matching tags):

| Ref | `package.json` version | SHA | Role |
|---|---|---|---|
| `dsh-v0.1.1-rc.2` | `0.1.1-rc.2` | `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e` | Official changelog base for 0.1.2-rc.1 |
| this checkout / brainstorm | `0.1.2-alpha.1` | `2c723c534aabf21517c9f3d2b7a4f5dec01c462f` | Prior EOD pin inventory ([PR #3](https://github.com/aa2246740/deepseek-harness/pull/3)) |
| `dsh-v0.1.2-rc.1` | `0.1.2-rc.1` | `a66e4702047846cdaa10c66c9d3df3951f5ea70d` | Requested upgrade |
| `dsh-v0.1.3-alpha.1` | `0.1.3-alpha.1` | `d347e703908d0406b7a7ef80e3a0e594d86b2215` | Not in scope; cited only as a later trap |

There is no tag named `v0.1.2-rc.1` on this fork or upstream. The exact upstream tag is **`dsh-v0.1.2-rc.1`**. Release notes: [v0.1.2-rc.1](https://github.com/deepseek-ai/deepseek-harness/releases/tag/dsh-v0.1.2-rc.1). Upstream also tagged `dsh-v0.1.2-alpha.2` through `dsh-v0.1.2-alpha.5` between this fork and rc.1; this fork has none of those tags. `CLAUDE_EVENTS` and the `KNOWN_SESSION_EVENT_TYPES` member set are unchanged on every one of them. `SessionEvent.ignorable` returns in alpha.2; `Session.events` is removed in alpha.4. No wanted pin hook appears and then vanishes.

The brainstorm was written on 2026-09-04 against this fork, after 0.1.2-rc.1 published on 2026-09-03. It already described 0.1.2-alpha.1 APIs, not 0.1.1-rc.2. The interesting window is therefore **alpha.1 → rc.1**, plus a sanity check of the official **0.1.1-rc.2 → 0.1.2-rc.1** changelog.

Standing product lean (unchanged): auto-draft plus one Pin/Keep confirm for **human-must-act work across sessions**. Not a classic todo app. Agent-only unfinished work stays in resume, `todo/write`, goals, plan review, and the session list.

-----

<a id="verdict"></a>
## Verdict

**Did 0.1.2-rc.1 add the hooks we wanted for EOD pin / human-must-act fold?**

**No.**

A generous reading of plugin-facing log APIs is at most **partial for implementation mechanics**, not for the wanted hooks:

- Claude Code `SessionEnd`, `FileChanged`, `Notification`, `TaskCompleted`, and `TeammateIdle` remain in the 23 unsupported events. Evidence: [`packages/hooks/hooks-claude-code/src/config.ts`](../packages/hooks/hooks-claude-code/src/config.ts) still enumerates only `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`, `SubagentStart`, and `SubagentStop`. The same list is identical on `dsh-v0.1.1-rc.2`, `dsh-v0.1.2-alpha.1`, `dsh-v0.1.2-rc.1`, and `dsh-v0.1.3-alpha.1`.
- Known Limitations still name those 23 events as ignored before group parsing: [hooks-claude-code Known Limitations](../packages/hooks/hooks-claude-code/README.md#known-limitations-and-deferred-work).
- Codex still maps five events only (`PreToolUse`, `PostToolUse`, `SessionStart`, `UserPromptSubmit`, `Stop`) in [`packages/hooks/hooks-codex/src/config.ts`](../packages/hooks/hooks-codex/src/config.ts).
- `KNOWN_SESSION_EVENT_TYPES` on rc.1 is identical to alpha.1. No `session/end`, `pin/*`, `notification/*`, or file-watch session events. The only SessionEventMap additions since 0.1.1-rc.2 are `model/selection`, `session-log-deepseek/delivery-accepted`, and `subagent/model-selection-policy` — none of them is an EOD digest.
- A `SessionEndState` in `packages/client/ui-trajectory` is a Trajectory compaction Chat node, not a Claude Code `SessionEnd` hook and not a desk digest.

Do not treat the RC release notes' "idle" items as user-away detection. The WebSocket heartbeat keeps idle **connections** alive. `agent/status` idle still means "no driver remains scheduled" after every turn.

-----

<a id="wanted-hooks"></a>
## Wanted hooks

| Wanted | On `dsh-v0.1.2-rc.1`? | Evidence |
|---|---|---|
| Claude Code / native `SessionEnd` | **No** | Unsupported list in [hooks-claude-code README](../packages/hooks/hooks-claude-code/README.md#known-limitations-and-deferred-work); `CLAUDE_EVENTS` in [`config.ts`](../packages/hooks/hooks-claude-code/src/config.ts); no `session/end` in `packages/core/session/src/known-event-types.ts` at that tag. `session/end-seed` remains a fork/resume log marker. |
| Calendar / EOD timer | **No** | Schedule is still session-local `after` / `at` / `every_seconds` (≥ 300s), no Cron, no email/push. Cold sessions stay overdue until resume. Header catalog in rc.1 is UI for those same records, not a desk clock. |
| Idle improvements for EOD | **No** | `agent/status` and `whenIdle()` / `runMaintenance()` existed before the RC. Heartbeat and connection-status UI are transport, not a debounce for "the human left for the day". |
| Cross-session `sessionQuery` for a host fold | **Already yes; unchanged for the pin job** | Host `listSessions()` / `filterSessions()` still have no cwd lock ([`packages/session-query/session-query/src/index.ts`](../packages/session-query/session-query/src/index.ts)). Model tools in [`workspace-access.ts`](../packages/session-query/tool-session-query/src/workspace-access.ts) still filter to exact caller `cwd`. `observeSession()` already existed in 0.1.2-alpha.1. |
| OS / product `Notification` hook | **No** | Claude Code `Notification` is still skipped. ACP/SDK "notification" payloads are protocol updates, not a desk push. |
| `FileChanged` | **No** | Still unsupported on the CC bridge. [`dsh-fs`](../packages/fs/fs/src/index.ts) still has `fs/write-intent`, `fs/edit-intent`, `fs/observed` per call — no watch primitive. |
| `TaskCompleted` | **No** | Still unsupported. Job completion still wakes an **in-session** idle owner; it is not a cross-session pin. |
| New Cordis events useful for a pin card | **No new digest event** | The pin-relevant `interface Events` names are the same between alpha.1 and rc.1. `tools/code-dispatch-log` was already `tools/ptc-dispatch-log` on alpha.1 (rename in the 0.1.1-rc.2 → rc.1 window), which does not help a pin. |

-----

<a id="what-012-rc1-actually-changed"></a>
## What 0.1.2-rc.1 actually changed

These are the plugin-adjacent deltas. None of them is a pin hook.

### Log read API (plugin must adapt on rc.1)

On `dsh-v0.1.2-rc.1`, `Session.events` is gone (removed in `dsh-v0.1.2-alpha.4`). Plugins that walked the full log now call `eventAt(seq)`, `snapshotEvents(from, toExclusive)`, `ownEvents()`, and `seq`. See `packages/core/session/src/index.ts` at that tag (this fork still has `get events()` because it is `0.1.2-alpha.1`). A `/pin` fold written against this checkout will not typecheck on rc.1 until it switches.

### `ignorable` restored (out-of-tree durable extras)

`0.1.2-alpha.1` (this fork / brainstorm) dropped `SessionEvent.ignorable`. `dsh-v0.1.2-alpha.2` restored it; rc.1 still has it (`packages/core/session/src/types.ts` at the tag). Unknown event types refuse reconstruction unless the envelope carries `ignorable: true`. That is a compatibility hatch for **out-of-tree** plugin events, not a first-class pin store: builds that do not know the type skip the event, and an in-tree required-on-read `pin/*` member would still join generated `KNOWN_SESSION_EVENT_TYPES`.

### Not EOD, do not misread

| rc.1 item | Why it is not the pin |
|---|---|
| `send_message` between parent and continuable child | Subagent follow-up, replacing one-way `report`. The tool package already existed in alpha.1. |
| Schedule catalog in the conversation header | Renders existing session-local `schedule/change` records. |
| WebSocket heartbeat | Keeps idle **sockets** alive (`packages/api/gateway`). Opposite of "the human went home". |
| Connection status / retry UI | Transport generation (`connection/reset`), not desk idle. |
| Trajectory `SessionEndState` | Compaction node in Trajectory, not session teardown. |

### Later tag, not this RC

`dsh-v0.1.3-alpha.1` (published 2026-09-04) is **not** 0.1.2-rc.1. It bumps session format to v2 (`SESSION_FORMAT_VERSION = 2`), replaces `assistant/chunk` with `assistant/attempt`, and makes persistence `SessionHandle`-scoped with asynchronous `agentLoop.create()`. A plugin targeting rc.1 must not assume those contracts. They would be a separate upgrade tax.

-----

<a id="plugin-without-core"></a>
## Plugin without core

**Yes**, the missing EOD *product* can still be a Cordis plugin without editing `agent-loop`. That was already true in the brainstorm. rc.1 does not change the legal extension points; it only changes how a plugin reads a Session log.

A native hook is an ordinary plugin on a documented extension point ([architecture](architecture.md), [extension cookbook](cookbook/extension-cookbook.md#a-hook-plugin-permission-gate-example)). The `packages/hooks` group remains a Claude Code / Codex compatibility adapter, not a product hook SDK.

### What a plugin may listen to today

Live Cordis events a pin plugin can use without a loop change (declarations on this tree, unchanged on rc.1 for these names):

| Event | Mode | Pin use |
|---|---|---|
| `agent/status` | emit | Closest quiet signal; debounce in plugin `Config` |
| `agent/session-start` | emit | Resume/startup only |
| `agent/turn-stopping` | serial | Stop-hook polarity (steer another step) — wrong for EOD |
| `agent/disposed`, `session/disposed` | emit | Process dropped the live agent, not 18:00 |
| `session/event` | emit | Observe `todo/write`, `goal/change`, `schedule/change`, `plan/mode`, approvals, questions |
| `session/created`, `session/flush` | emit / parallel | Boot / durability |
| `tools/pre-execute`, `tools/post-execute`, `tools/result` | waterfall / emit | High-volume; not FileChanged |
| `fs/write-intent`, `fs/edit-intent`, `fs/observed` | waterfall / emit | Per-call writes, no watch |
| `approval/request`, `user-questions/request` | waterfall | Human-must-act already blocking |
| `subagent/start`, `subagent/end` | emit | Child lifecycle |
| `commands/change` | emit | Registry churn, not a digest |

Host services a plugin may call (no cwd lock on the host API):

- `ctx.sessionQuery.listSessions()` / `filterSessions()` / `observeSession()` / `readSession()` / `readTitleSnapshots()`
- `ctx.commands` to register `/pin`
- `ctx.userQuestions.ask` for Pin / Keep / dismiss (plan-review `intent` already exists)
- `agent.whenIdle()` and `agent.runMaintenance(task)` to draft while public status stays `idle`
- `ctx.webhookRuntime` plus an **external** clock if someone insists on 18:00

The generated [event producer-consumer matrix](event-producer-consumer.md) is the exhaustive dispatcher/listener list for **this** checkout. It does not gain a SessionEnd row on rc.1.

### What a plugin may persist without core

- Log-only facts the harness already stores, folded at `/pin` time.
- Out-of-tree `ignorable` session events on rc.1 (skipped by readers that do not know the type).
- A Web Chat card via `ConversationNodeDefinition` is a **Client plugin**, not an `agent-loop` change. It is still product UI and out of this task's implementation scope.

Model-facing `session_search` remains cwd-locked. Do not ask the model in project A to search project B with shipped tools.

-----

<a id="still-requires-core"></a>
## Still requires core

These still need a new seam, a bridge mapping, or a loop/log vocabulary change — none of which 0.1.2-rc.1 shipped:

| Gap | Why a plugin cannot fake it honestly |
|---|---|
| `SessionEnd` / `session/end` | Bridges skip it; `session/disposed` is process teardown. Adding the CC mapping is a hooks-bridge change. |
| Calendar Cron / desk-wide 18:00 | Schedule has no calendar language and does not leave its session. |
| OS push / Claude Code `Notification` | No harness notify channel. Webhook creates a **new** root session from an external clock. |
| `FileChanged` watcher | No `fs.watch`. Instructions files reload on successful read/write/edit, not on external edits. |
| `TaskCompleted` as a desk event | Jobs wake the owning session only. |
| Cross-session todo / goal / schedule federation | Those records are last-write-wins **per session**. |
| Required-on-read in-tree `pin/*` | Joining `SessionEventMap` updates `KNOWN_SESSION_EVENT_TYPES` and persistence catalogs. |
| Waking a **cold** home session at 18:00 | Persistence will hold an overdue Schedule record; nothing pages the user. |

-----

<a id="product-lean"></a>
## Product lean

Product God wants auto-draft + Pin/Keep for human-must-act work **across sessions**, not another todo list.

rc.1 does not move that needle. The distinctive job is still folding **other / cold** sessions a human forgot. Host `sessionQuery` already lists them. Confirm UX already exists as `userQuestions.ask` (and plan-mode review). Todos in the open session remain the wrong store.

Recommended experiment, still optional, still not scheduled: a patch-layer plugin that registers `/pin`, folds host query, and asks Pin/Keep. Idle debounce and Schedule `at` may trigger the same fold later. Do not add `SessionEnd` for it. Do not wait for 0.1.2-rc.1 "hooks" that are not there.

-----

<a id="dev-note"></a>
## Dev Note

<details>
<summary>Working context — click to expand</summary>

Inspected by fetching `upstream` tags `dsh-v0.1.1-rc.2`, `dsh-v0.1.2-alpha.1` through `dsh-v0.1.2-alpha.5`, `dsh-v0.1.2-rc.1`, and `dsh-v0.1.3-alpha.1`. This fork's `master` is `2c723c534a` (`0.1.2-alpha.1` plus five local commits); merge-base with rc.1 is `cd5ef81481` (alpha.1); `git rev-list --count HEAD..dsh-v0.1.2-rc.1` is 656. Prior inventory: [PR #3 brainstorm](https://github.com/aa2246740/deepseek-harness/blob/cursor/eod-pin-brainstorm-a0fb/docs/eod-pin-brainstorm.md). Decision owner for "do not treat rc.1 as having added pin hooks": [Agent Note](../.agents/notes/proposed/feature/2026-09-05-eod-pin-hooks-0.1.2-rc1.md).

</details>
