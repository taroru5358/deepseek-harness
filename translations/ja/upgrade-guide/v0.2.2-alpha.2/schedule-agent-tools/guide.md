---
kind: upgrade-guide
description: "Schedule モデル tool はすべての Host task に対して `schedule_create`、`schedule_list`、`schedule_view`、`schedule_edit` になり、`schedule_update` と `schedule_delete` は削除される。"
---

# Schedule tool は RRULE と target ですべての Host task を管理する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/schedule-agent-tools/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/schedule-agent-tools/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.2-alpha.2/schedule-agent-tools/guide.md](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/schedule-agent-tools/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

以前は `@deepseek-ai/dsh-tool-schedule` が `schedule_create`、`schedule_list`、`schedule_update`、`schedule_delete` を登録していた。これらは呼び出し元 Session のリマインダーだけを管理し、`schedule_create` は `after_seconds`、`at`、`every_seconds`、`daily`、`weekly`、`cron` のいずれか1つの timing selector を取っていた。

現在このパッケージは `schedule_create`、`schedule_list`、`schedule_view`、`schedule_edit` を登録する:

- `schedule_create` は `title`、`prompt`、`rrule`（`DTSTART` 行と `RRULE` 行を各1行）、`target`（`this_session`、`new_once`、`new`）を取る。timing selector は受け付けない。
- `schedule_list` は active、paused、ended の task を、active なリマインダーの配列ではなく `{ items, shown, total, limit, offset }` の1ページとして返す。
- `schedule_view` は id で1つの task を読む。
- `action: "update"` を伴う `schedule_edit` は `schedule_update` を置き換え、`rrule` と `target` の変更や task の pause/resume も行える。`action: "delete"` は `schedule_delete` を置き換える。
- 各 tool は呼び出し元 Session の task だけでなく、id ですべての Host task を読み、変更し、削除する。
- 委譲された subagent からの呼び出しは、メッセージ `A delegated subagent cannot use scheduled tasks.` とともに `subagent_session` を返す。

既存のリマインダーは timing と delivery を維持する。記録済み Session は以前の呼び出しと結果を保持し、Web transcript は引き続きそれらを描画する。

影響:

- モデルに `schedule_update`、`schedule_delete`、または timing selector を呼ぶよう指示する prompt、skill、script
- 各 Session が自分のリマインダーだけを見ることに依存するデプロイ
- root エントリから `scheduleView` を、または `@deepseek-ai/dsh-schedule` の root もしくは `/client` エントリから `ScheduleView`、`ScheduleState`、`ScheduleDeliveryMode`、`ScheduleCreateValue`、`ScheduleListValue`、`ScheduleDeleteRequest`、`ScheduleDeleteValue`、`ScheduleTimingChange`、`ScheduleUpdateContent`、`ScheduleUpdateMiss`、`ScheduleUpdateRequest`、`ScheduleUpdateResult`、`ScheduleUpdateValue` を import する TypeScript コード

## 移行

1. prompt、skill、script で `schedule_update` と `schedule_delete` を `schedule_edit` に置き換え、各 timing selector を RRULE に置き換え、作成時に必須の `target` を指定する。たとえば `daily: { time: "09:00:00", time_zone: "Asia/Shanghai" }` は `rrule: "DTSTART;TZID=Asia/Shanghai:20261013T090000\nRRULE:FREQ=DAILY"` になる。呼び出し元 Session への delivery を維持するには `target: "this_session"` を使う。
2. preset の agent が他の Session の task を管理してはならない場合は、その preset から `@deepseek-ai/dsh-tool-schedule` 行を削除する。
3. TypeScript コードでは、書き込みに `ScheduleCreateRequest`、`ScheduleTiming`、`SchedulePatch`、`ScheduleEditRequest` を使い、読み取りと編集に `ScheduleViewResult`、`ScheduleListResult`、`ScheduleEditResult` を使う。`scheduleView`、`update`、`delete` の代わりに `ScheduleService.view`、ページ分割された `list`、`edit` を呼ぶ。`ScheduleDeleteResult`、`ScheduleDeleteChange`、`ScheduleDeleteInactiveResult` は root と `/client` から引き続きエクスポートされるが、上記の古い request/value 型はエクスポートされない。[Schedule サービス README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/schedule/schedule/README.md) がこれらの API を説明している。
4. カスタム構成では、この consumer をマウントする前に `tools` と `systemPrompt` の両方を提供する必要がある。登録は `schedule` も待つ。`tool:schedule` セクションが共有の authorization 命令を提供する。`complete: true` の prompt override はその命令自体を含めなければならない。
5. 変更した各 preset で Session を開き、tool リストに4つの tool 名が表示されることを確認する。task を委譲し、子からの Schedule tool 呼び出しが `subagent_session` を返すことを確認する。子の tool リストには4つの名前が含まれる。[preset を通じた subagent 構成の設定](../../v0.2.1-alpha.1/subagent-preset-composition/guide.md) に記載のとおり。
