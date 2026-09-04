# Agent Note: Defer a native end-of-day pin card

Status: proposed

English | [中文](2026-09-04-eod-pin-card.zh.md)

## Problem

A product idea asks the user, near end of day or on demand, to pin a card that lists today's work and unfinished work across parallel projects. Agent sessions already write a large log, and the obvious implementation temptations are a `SessionEnd` hook, a native Chat card, or a cross-session todo board. Coding agents in this repo already plan, execute, and resume from that log, so a second unfinished-work board can duplicate surfaces the harness already ships.

## Proposal

Do not add a native pin card, a `SessionEnd` hook, a `session/end` session event, or `pin/*` members of `SessionEventMap`. Do not change `agent-loop` for this idea. The evidence inventory of hooks, events, and APIs that actually exist, plus ranked designs, lives in [the EOD pin-card brainstorm](../../../../docs/eod-pin-brainstorm.md).

If a later experiment is warranted, build it as an **optional plugin**: register `/pin` on `ctx.commands`, fold other sessions with host `ctx.sessionQuery.listSessions()` / `filterSessions()` (no cwd lock), and confirm with `ctx.userQuestions.ask` (Pin / Keep / dismiss), copying plan-mode's existing review path. Idle (`agent/status` → `idle` with a config debounce) or Schedule `at` in one home session may trigger the same fold later. Do not implement that plugin in this change.

For agent-only coding, keep using resume, `todo/write`, goals, plan-mode review, and the session list. The only distinctive job a pin could add is folding **cold or other-cwd sessions** a human forgot; that job is already possible from host query without new lifecycle events.

## Related proposals

This note does not supersede [Task Surface](2026-08-04-task-surface.md) (a one-session structured form) or [interactive side sessions](2026-07-08-interactive-side-sessions.md) (fork plus merge-back). Neither lists other projects at end of day.

## Alternatives considered

- **Add Claude Code `SessionEnd` (or treat `Stop` as end of day):** rejected. The Claude Code and Codex bridges skip `SessionEnd`; `Stop` maps to `agent/turn-stopping` and **steers another step**. Native hooks are ordinary plugins on existing extension points ([interception extension points](../../implemented/feature/2026-06-30-interception-extension-points.md), [hook bridges](../../implemented/feature/2026-06-30-hook-bridges.md)).
- **Ship a native Chat card and `pin/*` events now:** deferred until a plugin experiment has a human-collaboration owner and evidence that the session list, todos, goals, and plan review are not enough. Durable product state would then follow [where new behavior goes](../../../../docs/architecture.md#where-new-behavior-goes).
- **Add a cross-session todo store, OS notifications, or calendar Cron inside Schedule:** rejected for this idea. Todos, goals, and Schedule are session-local; Schedule has no email/push and does not wake cold sessions. Those would be new seams, not a pin card.
- **Full-auto on every `agent/status` idle:** rejected as the default trigger. Idle fires after every turn; a debounce would be plugin `Config`, not a new core event, and still nags unless the user opted in.

## Acceptance criteria

- This change adds no `SessionEnd` mapping, no `session/end` event, no `pin/*` session events, and no product Chat card.
- A later pin plugin, if any, consumes `ctx.commands`, host `ctx.sessionQuery`, and `ctx.userQuestions` without editing `agent-loop`.
- Agent-only unfinished work continues to use resume, the last `todo/write`, goals, plan-mode review, and the session list rather than a second board.

## Risks

- Re-litigating `SessionEnd` because Claude Code names that hook; the inventory page is the cite-the-code answer.
- Treating "optional plugin later" as scheduled work; it is not. Ship nothing until a human-collaboration owner exists.
- A host fold over many session logs can be slow; any plugin must bound how many logs it opens and prefer headers plus latest `todo/write` / `goal/change` / `schedule/change` snapshots.
