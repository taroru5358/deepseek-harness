---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-10-07-title-reasoning-effort

日本語 | [English](../../../docs/persistence-changes/2026-10-07-title-reasoning-effort.md) | [中文](../../../docs/persistence-changes/2026-10-07-title-reasoning-effort.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-10-07-title-reasoning-effort.md](../../../docs/persistence-changes/2026-10-07-title-reasoning-effort.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

補助的な session-title リクエストのために選択された具体的な reasoning effort を記録する。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-10-07-title-reasoning-effort
baseline: false
changes:
  - root: "event:session/title-llm-request"
    previous: "2026-10-05-working-directory-attribution"
    after: "b33545b3244861bcbf73b407d24b53402710f5acacab7ae939537e5504a68dc6"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

新しい reasoningEffort フィールドはオプショナルである。既存の V4 レコードは有効なままで、記録された effort がないという元の状態を保つ。準備された title リクエストは、ルートが選択可能な reasoning を公開する場合に解決済みの effort を書き込む。Session 形式のバージョン引き上げは不要である。

<a id="verification"></a>
## 検証

対象を絞った LLM、アダプター、title-provider、リプレイのテストに合格した: 50 ファイル、1960 件のテスト。session-title-after-turn の SDK スナップショットは、デフォルトが high のルートから low の effort が選択されて記録された状態で合格した。

<a id="dev-note"></a>
## 開発メモ

なし。
