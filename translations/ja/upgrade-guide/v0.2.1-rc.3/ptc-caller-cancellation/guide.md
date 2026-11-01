---
kind: upgrade-guide
description: "呼び出し元がキャンセルした run_code の結果は CODE_RUN_FAILED ではなく ABORTED を使う。"
---

# PTC の caller cancellation は標準の tool エラーコードを使う

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.3/ptc-caller-cancellation/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.3/ptc-caller-cancellation/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.3/ptc-caller-cancellation/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.3/ptc-caller-cancellation/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

実行中の `run_code` 呼び出しが呼び出し元によってキャンセルされ、runtime が `abort` を報告すると、その `tool/result` エラーは `CodeRunFailedError` / `CODE_RUN_FAILED` から `AbortError` / `ABORTED` に変わる。結果の content は `Error: tool call aborted` になり、runtime の診断、捕捉された出力、sandbox の要約を置き換える。これらの結果を分類する SDK client と Session consumer が影響を受ける。

Agent は引き続き `turn/end` に具体的なキャンセル原因を記録し、hook の `reason` も含む。timeout や cleanup 失敗などの runtime 失敗、および呼び出し元のキャンセルを伴わない runtime ローカルのキャンセルは `CODE_RUN_FAILED` を維持する。キャンセルは引き続き cleanup を待つ。完了した tool の効果をロールバックせず、保持された runtime 変数はクリアされる場合がある。

## 移行

1. 新しい `tool/result` event の consumer では、`data.error.code === 'ABORTED'` で caller cancellation を分類する。`data.reason.kind === 'aborted'` の場合、Agent のキャンセル原因は対応する `turn/end` event の `data.reason.reason` から読む。
2. 履歴上の PTC キャンセル結果を読む際は `CODE_RUN_FAILED` のサポートを維持する。既存の Session log は書き換えられない。
3. 実行中の `run_code` 呼び出しをキャンセルし、その結果の code が `ABORTED` で、元の Agent キャンセル原因を含む `turn/end` event が続くことを確認する。通常の runtime 失敗が引き続き `CODE_RUN_FAILED` を報告することを確認する。
