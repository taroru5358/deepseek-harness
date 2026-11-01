---
kind: upgrade-guide
description: "Auto review Session はアプリケーション終了時に permission preset を保持し、再度開くときに統合を必要とする。"
---

# シャットダウン後に Auto review Session を再度開く

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/auto-review-shutdown/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/auto-review-shutdown/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/auto-review-shutdown/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/auto-review-shutdown/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

アプリケーションのシャットダウンは以前、disposal の順序によって、ライブな Auto review Session を Full access に変更することがあった。シャットダウンは現在、Auto とその approval policy を保持する。実行中のアプリケーションから Auto review plugin を削除すると、そのライブな Auto Session は引き続き Full access に変更される。

保存済みの Auto Session は、再度開く前に Auto review 統合を必要とする。これは、headless 呼び出しが別の permission preset を要求する場合にも適用される。復元は新しいフラグを適用する前に保存済みの選択を検証する。既存の policy レコードは書き換えられない。

## 移行

1. Session を再度開く profile で Auto review 統合を有効に保つ。一度限りの `--patch` overlay でロードした場合は、resume 時にその overlay を再度渡す。
2. 保存済みの Auto Session を別の preset に変更するには、まず統合を存在させた状態で再度開き、次に新しい preset を選択する。headless 呼び出し元は同じ Auto overlay とともに `--session-id <id> --permission-mode <preset>` を指定できる。
3. 統合を存在させた状態で Session を閉じて再度開く。明示的に変更しない限り、選択された preset が Auto のままであることを確認する。
