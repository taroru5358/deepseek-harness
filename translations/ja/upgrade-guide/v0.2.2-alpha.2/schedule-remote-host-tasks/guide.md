---
kind: upgrade-guide
description: "Schedule client は Host 全体の単一管理 API と型付き timing レコードを使い、保存済み Host task は既存の domain とレイアウトを維持する。"
---

# Schedule は Host 全体の単一管理 API を使う

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/schedule-remote-host-tasks/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/schedule-remote-host-tasks/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.2-alpha.2/schedule-remote-host-tasks/guide.md](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/schedule-remote-host-tasks/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

Schedule 管理は id ですべての Host task を扱う。ネイティブ API と Remote API は1つのサービスを共有し、以前の Session バインドの `delete` と `update` メソッドは削除された。ネイティブ作成は型付きの `timing` を取り、catalog 結果は `kind` で After、At、Every、Daily、Weekly、Cron、RRULE を判別する。

既存の `schedule` domain は version 2 を書き込み、起動時の書き込みなしで version 1 を読む。route、envelope、layout は維持される。SQLite の物理スキーマは version 1 のままである。アップグレード前に store をバックアップする。最初に成功した v2 書き込みが unit を v2 にスタンプし、v1 バイナリはそれを開くことを拒否する。ダウングレードにはそのバックアップの復元が必要である。編集と delivery は別個の移行コマンドなしで現在の行を保存する。plugin README が[保存スキーマ、アップグレードのタイミング、コスト](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/schedule/schedule/README.md#storage-format)を所有する。

## 移行

1. ネイティブ呼び出し元は `create({ title, prompt, timing, target? }, actorSessionId?, signal?)` と `edit(request, actorSessionId?, signal?)` を使う。Remote 呼び出し元は `request` 引数付きで `schedule/create` と `schedule/edit` を使う。Cron selector は `{ kind: "cron", cron: { expression, time_zone } }` として、RRULE は `{ kind: "rrule", rrule }` としてラップする。モデル tool の引数は引き続き RRULE を公開する。
2. Session バインドの削除を `edit({ id, action: "delete" })` に置き換え、更新を `edit({ id, action: "update", patch, expectedUpdatedAt? })` に置き換える。timing の編集には `patch.timing` を使う。read revision を渡すか、revision がない場合の比較には `null` を渡す。`schedule_conflict` は再読み取りで処理する。
3. `list({ active?, limit?, offset? })` は Host 全体で `{ items, shown, total, limit, offset }` を返す。`catalog()` はすべての task を返す。`view({ id })`、`history({ id, limit, before? })`、`run({ id })` は owner Session を必要としない。再利用可能な destination は `target` から、各 receipt の destination は `targetSessionId` から読む。
4. パッケージ root または `/client` がエクスポートする接頭辞なしの型（`ScheduleRecord`、`ScheduleTiming`、`ScheduleCatalogEntry` など）を使う。timing フィールドを読む前に `kind` で絞り込む。`inactive` の代わりに `paused` と `ended` を処理し、作成・変更タイムスタンプの欠落を許容する。root helper の `scheduleView`、`decodeScheduleChange`、`foldScheduleEvents`、`SCHEDULE_CHANGE_VERSION` は削除された。履歴 event 型は凍結されたデータ宣言として残る。[tool アップグレードガイド](../schedule-agent-tools/guide.md) が削除された型とその代替を列挙している。
5. Automation tasks が既存の Cron とカレンダー task を表示し、メタデータ編集がその timing と履歴を保持することを確認する。履歴 Session-log リマインダーは引き続き明示的な再作成を必要とする。それらはこの Host-task 変更の範囲外である。
