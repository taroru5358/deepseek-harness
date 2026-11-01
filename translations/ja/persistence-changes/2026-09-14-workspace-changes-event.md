---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-09-14-workspace-changes-event

日本語 | [English](../../../docs/persistence-changes/2026-09-14-workspace-changes-event.md) | [中文](../../../docs/persistence-changes/2026-09-14-workspace-changes-event.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-09-14-workspace-changes-event.md](../../../docs/persistence-changes/2026-09-14-workspace-changes-event.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

トップレベルのターンが変更したファイルを記録する、ログ専用の workspace/changes イベントを追加する。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-09-14-workspace-changes-event
baseline: false
changes:
  - root: "event:workspace/changes"
    previous: null
    after: "e308ccf867a5398e316e0af8cb6ce238a8d33a63b9b384c8250a686786285f72"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

同じ Session 形式バージョンにおける新しいルートである。既存のログにはそのようなイベントが含まれず、有効なままである。それ以前のリーダーは、読み取り時に必須のあらゆるイベントと同様に、それを含むログを拒否する。このイベントは Web バンドルの workspace-changes プラグインだけが追記し、モデルには決して見えない。Web の変更ファイル・カードが唯一のコンシューマーであり、ターンごとに最新のイベントを読む。

<a id="verification"></a>
## 検証

pnpm exec vitest run packages/deliverables/workspace-changes packages/client/ui-deliverables: 165 件のテストに合格した。snapshots/web/changed-files-turn は記録されたイベントを Web プロファイル経由でリプレイする。

<a id="dev-note"></a>
## 開発メモ

なし。
