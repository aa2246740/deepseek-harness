# End-of-day pin card — hooks inventory and feasibility

English | [中文](eod-pin-brainstorm.zh.md)

Discussion and feasibility only. This page is not a product contract and does not describe shipped behavior beyond the inventory of existing hooks, events, and APIs.

## Summary

DeepSeek Harness already records today's work in per-session event logs, titles, todos, goals, plan-mode reviews, approvals, questions, jobs, and reminders. It has no `SessionEnd` hook, no calendar end-of-day timer, no cross-session todo list, and no OS notification channel. A native "pin today's work" card would mostly duplicate those surfaces for agent-only coding, and would only earn its keep as an optional host plugin that folds **other sessions** for a human who must act.

## Table of Contents

- [Hooks inventory](#hooks-inventory)
- [Feasibility](#feasibility)
- [Brainstorm](#brainstorm)
- [When useful](#when-useful)
- [Verdict](#verdict)
- [Dev Note](#dev-note)

-----

<a id="hooks-inventory"></a>
## Hooks inventory

This inventory cites APIs that exist in this repository. It does not invent Claude Code or Codex events that the bridges skip.

### What "hooks" means here

A **native hook** is an ordinary Cordis plugin on a documented extension point. There is no separate native-hooks package. That rule is recorded in the [interception extension-points Agent Note](../.agents/notes/implemented/feature/2026-06-30-interception-extension-points.md) and restated in the [extension cookbook](cookbook/extension-cookbook.md#a-hook-plugin-permission-gate-example).

The `packages/hooks` group is a **compatibility adapter** for existing Claude Code and Codex `hooks.json` command hooks. Bespoke product behavior belongs on the same Cordis events the bridges already program against, not on a new `hooks.json` dialect. See the [hooks group map](../packages/hooks/README.md), [hook-protocol](../packages/hooks/hook-protocol/README.md), [hooks-claude-code](../packages/hooks/hooks-claude-code/README.md), and [hooks-codex](../packages/hooks/hooks-codex/README.md).

The architecture map of where new behavior goes is [architecture.md](architecture.md#where-new-behavior-goes). The generated [event producer-consumer matrix](event-producer-consumer.md) lists every harness event's dispatchers and listeners.

### Claude Code and Codex hook points that actually fire

Both bridges map a small command-hook subset onto harness events. Shared durable records are log-only `hook/invoked` and `hook/result`, turn-enclosed, declared in [`packages/hooks/hook-protocol/src/events.ts`](../packages/hooks/hook-protocol/src/events.ts). `SessionStart` runs before turn 1 and does **not** append `hook/*`; it injects context instead.

| External hook | Harness extension point | What it can do today | Pin-card relevance |
|---|---|---|---|
| `SessionStart` | `agent/session-start` emit | Attach context; detached, can miss the first request | Session boot / resume, not end of day |
| `UserPromptSubmit` | `agent/pre-step` waterfall | Block the prompt or attach context | On-demand pin if the user types a prompt, not idle EOD |
| `PreToolUse` | `tools/pre-execute` waterfall | Deny (Claude Code can `ask`) | Tool gating, not a digest |
| `PostToolUse` | `tools/post-execute` waterfall | Block with feedback or attach context | Could observe writes, but fires on every tool |
| `Stop` | `agent/turn-stopping` serial | Force **another** model step via `steer()` | Opposite of "day is over" |
| `SubagentStart` / `SubagentStop` | `subagent/start`, `subagent/end` | Claude Code only; stop is observe-only | Child lifecycle, not a desk digest |

`{"continue": false}` is folded into `hook/result` and has **no run-level halt**. There is no `SessionEnd` mapping. Claude Code's `FileChanged`, `Notification`, `TeammateIdle`, `TaskCompleted`, `PreCompact`, `PostCompact`, and `SessionEnd` are listed as unsupported and ignored at parse time in [hooks-claude-code Known Limitations](../packages/hooks/hooks-claude-code/README.md#known-limitations-and-deferred-work). Codex likewise drops `PermissionRequest`, `PreCompact`, `PostCompact`, `SubagentStart`, and `SubagentStop` ([hooks-codex](../packages/hooks/hooks-codex/README.md#known-limitations-and-deferred-work)).

### Agent and session lifecycle (the real native points)

Declared in [`packages/core/agent/src/runtime-types.ts`](../packages/core/agent/src/runtime-types.ts) and [`packages/core/session/src/index.ts`](../packages/core/session/src/index.ts). Turn flow is [architecture.md](architecture.md#turn-flow) and [agent-lifecycle.md](agent-lifecycle.md).

| Event or API | Mode | When it runs | Pin-card relevance |
|---|---|---|---|
| `agent/created` | emit | Live agent published | Too early |
| `agent/session-start` | emit | Once before turn 1; `source` is `startup` \| `resume` \| `clear` \| `compact` | Resume is real; not EOD |
| `agent/status` | emit | `idle` ⇄ `running`; idle means no driver remains scheduled | Closest "session went quiet" signal; fires after **every** turn |
| `agent.whenIdle()` | method | Quiescence after the current whole-agent activity | Same as idle; used by [schedule](../packages/schedule/schedule/README.md) |
| `agent.runMaintenance(task)` | method | One non-turn task from true idle; public status stays `idle` | Safe place to draft a digest without opening a turn |
| `agent/pre-step` | waterfall | Every proposed step | Can inject context; not an EOD timer |
| `agent/turn-stopping` | serial | Model owes no response; listeners may `steer()` to continue | Stop-hook semantics, not pin |
| `agent/disposed` | emit | Agent left the registry after quiescence, before session detach | Process teardown, not end of day |
| `session/created` | emit | Session published into the live store | Boot |
| `session/event` | emit | Every append | Universal observe point for logs, todos, tools, docs-as-writes |
| `session/flush` | parallel | Durability checkpoint | Persistence, not UX |
| `session/disposed` | emit | Live store entry left | Closing a tab/process, not calendar EOD |

`SessionStartSource` is `'startup' | 'resume' | 'clear' | 'compact'` ([`runtime-types.ts`](../packages/core/agent/src/runtime-types.ts)). Resume is `ctx.agents.resume({ resumeSessionId })` after `ctx.sessionPersistence.load` ([session-persistence](../packages/session/session-persistence/README.md), [agent registry](../packages/core/agent/src/index.ts)).

There is **no** `session/end` session event and **no** harness `SessionEnd` hook. `session/end-seed` is a fork/resume seed-boundary marker on the log, not process teardown or calendar end. LLM idle watchdogs in adapters are stream read timeouts, not user-away detection.

### Durable session facts a digest could fold

The session log is the source of truth ([subsystems/session.md](subsystems/session.md), [persistence catalog](persistence-catalog.md)). Per-session state is last-write-wins on replay; it is not a cross-project board.

| Fact | Event / API | Scope | Notes |
|---|---|---|---|
| Turns, messages, tool calls | `turn/*`, `user/message`, `assistant/message`, `tool/call`, `tool/result` | One session | Full transcript of today's work |
| Title | `session/title` via [`dsh-session-title`](../packages/session/session-title/README.md) | One session | List-row name; user rename pins against auto-refresh |
| Todos | `todo/write` via [`dsh-tool-todo`](../packages/todo/tool-todo/README.md) | One agent session | Whole-list replace; **not shared** across agents; UI projection **clears when the next turn starts** |
| Plan mode | `plan/mode` via [`dsh-plan-mode`](../packages/plan/plan-mode/README.md) | One agent | Soft guidance; `exit_plan_mode` already presents Approve / Keep planning through [user-questions](../packages/interaction/user-questions/README.md) |
| Goals | `goal/change` via [`dsh-goal`](subsystems/goal.md) | Same session | Phases `active` \| `paused` \| `blocked` \| `complete`; [goal-round-driver](../packages/goal/goal-round-driver/README.md) auto-continues while idle and armed |
| Schedule | `schedule/change` via [`dsh-schedule`](subsystems/schedule.md) | Same session | `after` / `at` / `every_seconds` (≥ 300s); **no calendar/Cron**; **no email/push**; cold sessions stay overdue until that session is resumed |
| Approvals | `approval/asked`, `approval/decided` | Open turn | Human-must-act already blocking |
| Commands | `command/*` | One session | `/plan`, `/goal`, `/compact` run without a model message |
| Compaction | `compaction/*` | One session | History summary, not a daily pin |
| Seed boundary | `session/end-seed` | One session | Fork/resume marker; **not** SessionEnd or calendar EOD |
| Workflow runs | `tool-workflow/*`, `workflow/*` live events | One session / engine | Foreground orchestration |
| Experimental team board | `team/task` | One session | Private experimental package, not a release surface |

Filesystem writes are not a "docs hook". [`dsh-fs`](../packages/fs/fs/README.md) has no watch primitive. `fs/write-intent`, `fs/edit-intent`, and `fs/observed` are per-call gates ([fs-observation-policy](../packages/fs/fs-observation-policy/README.md)). [`dsh-agent-instructions`](../packages/context/agent-instructions/README.md) loads `AGENTS.md` / `CLAUDE.md` on successful read/write/edit or resume baseline — **there is no file watcher**. [`dsh-skill-filesystem`](../packages/skill/skill-filesystem/README.md) watches skill roots only.

### Cross-session APIs (the only "parallel projects" path)

Todos, goals, plan, and schedule are **session-local**. The host can still see many sessions:

| API | Package | What it returns | Authorization |
|---|---|---|---|
| `ctx.sessionPersistence.list()` | [session-persistence](../packages/session/session-persistence/README.md) | Stored headers | Host |
| `ctx.sessionQuery.listSessions()` / `filterSessions()` | [session-query](../packages/session-query/session-query/README.md) | Live-preferred corpus; filter by cwd, created-at, parent | Host; no cwd lock |
| `ctx.sessionQuery.searchSessions()` | [session-query-sqlite](../packages/session-query/session-query-sqlite/README.md) | Ranked FTS | Host; opt-in index |
| `sessionController.list()` | [session-controller](../packages/api/session-controller/README.md) | Summaries with titles, `running`, `lastPromptAt`; does not activate agents | Host / Web list |
| `session_search` and siblings | [tool-session-query](../packages/session-query/tool-session-query/README.md) | Model-facing search | **Exact `cwd` match**; not mounted in shipped host compositions |
| `ctx.sessionReferenceResolver.listCandidates()` | [session-reference](../packages/context/session-reference/README.md) | Other sessions ranked by cwd affinity for `@` mentions | Host mention UX |

A model running in project A cannot use the shipped session-query **tools** to read project B. A host plugin or slash command **can**.

### Human-gate and clock seams already on the product

| Seam | Role | Already a "pin"? |
|---|---|---|
| [`dsh-user-questions`](../packages/interaction/user-questions/README.md) + [`ask_user_question`](../packages/interaction/tool-ask-user/README.md) | Pause until the human answers | Yes, blocking |
| [`dsh-user-approval`](../packages/interaction/user-approval/README.md) | One-shot allow/reject for a tool | Yes, blocking |
| Plan-mode `exit_plan_mode` | Approve / Keep planning | Yes, structured confirm |
| [`dsh-commands`](../packages/interaction/commands/README.md) | `/name` without a model turn | Natural `/pin` mount |
| [`dsh-schedule`](../packages/schedule/schedule/README.md) | Session-local reminder as a later user message | Closest clock; not desk-wide |
| [`dsh-time-context`](../packages/context/time-context/README.md) | Per-step clock text for the model | Not a timer |
| [`dsh-tool-jobs`](../packages/jobs/tool-jobs/README.md) | Wake idle owner on job completion | Work unfinished, in-session |
| [`dsh-webhook`](../packages/webhook/webhook/README.md) | External delivery → new root session | Needs an **external** clock |
| Proposed [Task Surface](../.agents/notes/proposed/feature/2026-08-04-task-surface.md) | Declarative one-session form; `show_task_surface` ends the turn | Overlaps "Pin/Keep" UI if that proposal ships; not an EOD digest |

Web Chat cards for a new durable fact would register a `ConversationNodeDefinition` ([conversation subsystem](subsystems/conversation.md)). That is product UI, out of scope for this page.

-----

<a id="feasibility"></a>
## Feasibility

Feasibility assumes a digest of "today's work + unfinished work across parallel projects", with Product God leaning auto-draft plus one Pin/Keep confirm. Ratings: **easy** (plugin on existing events/APIs), **needs plugin** (new package, no loop change), **needs core** (new lifecycle event, loop change, or new cross-session store).

| Candidate feed | Exists today? | How a plugin would use it | Rating | Gap |
|---|---|---|---|---|
| Session end / process exit | `session/disposed`, `agent/disposed` | Observe teardown | easy | Means "this process dropped the live agent", not 18:00 |
| Idle after a turn | `agent/status` idle, `whenIdle`, `runMaintenance` | Draft while status stays idle | easy | Fires after every turn; needs debounce/clock the harness does not provide as EOD |
| Plan complete | `exit_plan_mode` + `plan/mode` | Already a review | easy | Only that session, only while planning |
| Tool results / doc writes | `session/event` `tool/result`, `fs/write-intent` | Tally writes | easy | High volume; no `FileChanged` watcher; instructions files have no watch |
| Claude Code `Stop` | `agent/turn-stopping` | Would **continue** the run | easy | Wrong polarity for EOD |
| Claude Code `SessionEnd` | **No** | — | needs core | Bridges explicitly skip it; do not add it for a pin card |
| Calendar 18:00 | Schedule `at` with `{ date, time, time_zone }` | Reminder in **that** session | needs plugin | Session-local; cold session waits for resume; no OS notify |
| Recurring daily | Schedule `every_seconds` ≥ 300 | Fixed-rate from creation, not wall-clock 18:00 | needs plugin | Not calendar-aligned; still session-local |
| Cross-project list | `sessionQuery.listSessions` / `filterSessions` created-at | Host fold titles, last `todo/write`, goal phase | easy | Model tools cannot cross cwd; host/command can |
| Shared unfinished-work store | **No** | New `SessionEventMap` or sidecar DB | needs core | Todos/goals/schedule do not federate |
| Desk-wide push | **No** | Webhook + external cron, or OS notify | needs plugin + external clock | Webhook creates a **new** session; schedule never leaves its session |

Adding `SessionEnd`, a process-level daily timer, or a cross-session todo board would be a core (or new-seam) change. None of those is required to ship a `/pin` command that reads logs the harness already stores.

-----

<a id="brainstorm"></a>
## Brainstorm

Designs ranked by fit to **hooks and APIs that exist**, not by visual appeal. None of these is scheduled work.

### 1. On-demand `/pin` command + host query fold + Pin/Keep question — best fit

A plugin registers `/pin` on [`ctx.commands`](../packages/interaction/commands/README.md). The handler calls `ctx.sessionQuery.listSessions` / `filterSessions` (created-at today, optional cwd list), loads each log or projection, and folds titles, last `todo/write`, goal phase, plan-mode flag, and overdue `schedule` records. It then calls `ctx.userQuestions.ask` with Approve-style options (Pin / Keep / dismiss), copying plan-mode's existing confirm path. Durable keep, if wanted, is a log-only event on a chosen "desk" session or a small projection unit — still a plugin, not a loop change.

This matches Product God's auto-draft + one confirm **without inventing EOD**. The user (or a later idle trigger) decides when to run it. Cross-project works because the host query is not cwd-locked.

### 2. Idle auto-draft in the live session

Listen to `agent/status` → `idle`, then `runMaintenance` to build the same fold and `inject` or `followup` a draft, then `ask`. Easy plugin. The harness idle signal is **per turn**, so a naive listener would nag after every reply. A config debounce (hours, not seconds) is required; that debounce is a plugin `Config` field, not a new core event.

### 3. Schedule `at` local evening in one home session

Use the existing [Schedule overlay](user/guide/schedule.md): `schedule_create` with `LocalAtInput` at 18:00 in the browser zone. Delivery is an ordinary follow-up in **that** session after `whenIdle`. Easy overlay/plugin. Parallel projects are invisible unless the home session's plugin also reads `sessionQuery`. Cold home sessions store the overdue reminder until resume; nothing reaches the user's phone.

### 4. Reuse shipped list + todos + goals + plan review — already shipped

The Web session list already shows titles, running state, and last prompt time ([session-controller list](../packages/api/session-controller/src/list.ts)). Todos, goals, and plan review already surface unfinished in-session work. This design is documentation and UX emphasis, not a new card. It is the strongest answer to Cola's doubt for **agent-only** work.

### 5. External clock → webhook → digest session

[`ctx.webhookRuntime`](../packages/webhook/webhook/README.md) can create a new root session from a trusted delivery. An OS cron or calendar app at 18:00 is the clock DSH does not own. The new session's prompt can instruct the model to summarize; host-side folding is still more honest than hoping the model searches other cwds (shipped `session_search` refuses them). Needs a plugin rule plus an external scheduler.

### 6. Treat `exit_plan_mode` as the pin

Plan mode already stops for Approve / Keep planning. Useful when the human must bless a plan before execution. Useless as "today across projects" and unused when the agent is not in plan mode.

### 7. Claude Code `Stop` / hoped-for `SessionEnd` in `hooks.json`

`Stop` forces another step. `SessionEnd` is not implemented. Using the compatibility bridges as a product pin would fight their documented limits ([hook bridges Agent Note](../.agents/notes/implemented/feature/2026-06-30-hook-bridges.md)). Reject for this idea.

### 8. New `pin/*` events + Chat card + core store

Extend `SessionEventMap`, add a projection, register a `ConversationNodeDefinition`. That is the architecture-legal way to add durable product state ([architecture.md](architecture.md#where-new-behavior-goes)). It needs core-adjacent packages and Web UI this task is not implementing. The proposed [Task Surface](../.agents/notes/proposed/feature/2026-08-04-task-surface.md) already covers a one-session structured confirm; it does not list other projects. Do not start this unless a plugin experiment (design 1) proves a human-collaboration owner.

-----

<a id="when-useful"></a>
## When useful

Cola’s doubt is right for the default coding loop. Product God’s card is useful only in a narrower slice.

### Agent-only work (plan, execute, resume) — mostly useless

A coding agent already:

- writes a standing plan with `todo_write` (log-backed; resume reconstitutes it);
- enters `/plan`, then `exit_plan_mode` for a human gate, or skips planning and just runs;
- continues a same-session objective through `ctx.goals` and the idle round driver;
- wakes on job completion and on due schedule records while the session is live;
- reconstitutes the whole log on `ctx.agents.resume`.

Unfinished agent work is **in the log**. The next session does not need a pin to know what to do. A pin card that lists "still pending todos" duplicates `todo/write`. A pin that lists "files the agent wrote" duplicates git and the transcript. Agents that execute immediately after planning do not wait for a morning pin.

The todo **UI** projection clearing on the next turn ([todo README](../packages/todo/tool-todo/README.md)) is a standing-plan lifetime choice, not proof that unfinished work vanished — the last `todo/write` remains in the log and on resume.

### Human-must-act work — already blocking, weak extra pin

These already stop the loop or the tool:

- `ask_user_question` / `user-questions/request`
- `approval/request`
- plan-mode review (`intent: { kind: 'plan-review' }`)
- `goal` phase `blocked`

Pinning them on a second card would duplicate an unanswered question. The useful remainder is **work the human forgot that is not currently on screen**: another session's blocked goal, an overdue reminder on a **cold** session (Schedule will not notify outside that session), or a colleague-facing doc the agent wrote in a different cwd.

### Cross-project / human collaboration — the only distinctive job

"Parallel projects" in this repo are parallel **sessions**, usually with different `SessionHeader.cwd`. There is no shared todo. Experimental agent-teams share a board **inside one session** and are excluded from official releases.

The distinctive job is: at a human-chosen time, fold **many** session headers and latest log-only snapshots into one checklist the human can keep or dismiss. That job cannot be done by `todo_write` in the session the user happens to have open. It **can** be done by host `sessionQuery` today. It does not need `SessionEnd`.

### Auto vs manual vs choice

| Trigger | Honest mapping | Risk |
|---|---|---|
| Full-auto at 18:00 | Not a harness hook; Schedule in one session or external cron | Misses cold sessions; no push |
| Full-auto on idle | `agent/status` idle | Nags every turn unless debounced |
| Manual `/pin` | `ctx.commands` | Matches "on demand"; no fake clock |
| Auto-draft + Pin/Keep | Command or debounced idle, then `userQuestions.ask` | Product God lean; still a plugin |

Recommend **manual or user-chosen auto-draft**, not a new `SessionEnd`.

-----

<a id="verdict"></a>
## Verdict

**Do not build a native pin card in core, and do not add a `SessionEnd` hook for it.**

Ship nothing in `agent-loop`. The loop already exposes idle, turn-stopping, resume, and the session log. A pin product would be a **Consumer** of those points, not a new driver.

**If a later experiment is warranted, build it as an optional plugin** (command + `ctx.sessionQuery` fold + `ctx.userQuestions` confirm), mounted from a patch or bundle, not as a Claude Code `hooks.json` feature. Idle or Schedule can trigger the same fold later. Durable `pin/*` events and a Chat card are justified only after that plugin has a human-collaboration owner and evidence that session list + todos + goals + plan review were not enough.

**Do not build** a cross-session todo store, OS notifications, or calendar Cron inside Schedule to unblock this idea. Those would be new seams with their own consumers, not a pin card.

Cola’s default: for agent-only coding, use resume, todos, goals, and the session list. Product God’s confirm UX already exists as plan-mode review and `ask_user_question`; reuse those instead of a new card type.

-----

<a id="dev-note"></a>
## Dev Note

<details>
<summary>Working context — click to expand</summary>

This page is the evidence inventory and ranked designs. The standing decision (no native pin, no `SessionEnd`, plugin-only if a later experiment has a human-collaboration owner) lives in the [EOD pin-card Agent Note](../.agents/notes/proposed/feature/2026-09-04-eod-pin-card.md). Related live proposals that overlap UI, not EOD: [Task Surface](../.agents/notes/proposed/feature/2026-08-04-task-surface.md) (structured one-session form) and [interactive side sessions](../.agents/notes/proposed/feature/2026-07-08-interactive-side-sessions.md) (fork + merge-back). Neither replaces a cross-session digest. Implementing design 1 should name the plugin package and any `SessionEventMap` members in a new note that supersedes that deferral.

</details>
