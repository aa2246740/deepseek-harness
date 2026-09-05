# Agent Note: 0.1.2-rc.1 did not add EOD pin hooks

Status: proposed

English | [中文](2026-09-05-eod-pin-hooks-0.1.2-rc1.zh.md)

## Problem

Product God asked whether DeepSeek Harness 0.1.2-rc.1 added `SessionEnd`, a calendar end-of-day timer, idle-as-away, OS notification, `FileChanged`, `TaskCompleted`, or other Cordis events that would unblock an end-of-day pin / human-must-act fold. Treating the RC as having those hooks would send a plugin or core change down the wrong path. The prior inventory lives on [PR #3](https://github.com/aa2246740/deepseek-harness/pull/3) and was written against this fork's `0.1.2-alpha.1` checkout. Intermediate upstream tags `dsh-v0.1.2-alpha.2` through `alpha.5` do not add the wanted hooks either.

## Proposal

Do not treat official tag `dsh-v0.1.2-rc.1` as having added those hooks. The evidence page is [the 0.1.2-rc.1 hooks delta](../../../../docs/eod-pin-hooks-0.1.2-rc1.md).

Keep the standing product lean: an optional Cordis plugin (`/pin` + host `ctx.sessionQuery` + `ctx.userQuestions.ask`), not a native card, not a Claude Code `SessionEnd`, not a todo app. A plugin targeting rc.1 must read logs with `eventAt` / `snapshotEvents`, not `Session.events`. Out-of-tree `ignorable` events return in alpha.2 and remain on rc.1, but they are not a first-class pin store.

This note does not implement the plugin. It does not supersede [Task Surface](2026-08-04-task-surface.md) (a one-session structured form).

## Alternatives considered

- **Treat rc.1 "idle" / heartbeat / schedule header / `send_message` as the pin:** rejected. Those are connection keepalive, session-local schedule UI, and subagent follow-up.
- **Wait for `SessionEnd` before experimenting:** rejected. The Claude Code bridge still skips it on rc.1 and on `dsh-v0.1.3-alpha.1`.
- **Assume this fork is 0.1.2-rc.1:** rejected. This checkout is `0.1.2-alpha.1`, 656 commits behind the tag.
- **Assume a wanted hook landed in `dsh-v0.1.2-alpha.2`–`alpha.5` and was removed before rc.1:** rejected. Those tags keep the same `CLAUDE_EVENTS` list and `KNOWN_SESSION_EVENT_TYPES` members.

## Acceptance criteria

- The inventory page names `dsh-v0.1.2-rc.1` as the exact tag, states **No** for the wanted hooks, and cites paths.
- No `SessionEnd` mapping, `session/end` event, or pin Chat card ships in this change.
- A later pin plugin, if any, still consumes `ctx.commands`, host `ctx.sessionQuery`, and `ctx.userQuestions` without editing `agent-loop`.

## Risks

- Re-litigating `SessionEnd` from the RC changelog's unrelated "idle" wording.
- Writing a `/pin` fold against this fork's `Session.events` getter and discovering it does not exist on rc.1.
- Confusing `dsh-v0.1.3-alpha.1` `SessionHandle` / format v2 with this RC.
