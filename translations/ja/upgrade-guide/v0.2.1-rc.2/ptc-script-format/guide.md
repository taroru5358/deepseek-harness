---
kind: upgrade-guide
description: "PTC の programFormat は Script 構文を含み、consumer は cross-call memory のために retainsState を読む。"
---

# Script プログラムと明示的な state retention を扱う

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/ptc-script-format/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/ptc-script-format/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/ptc-script-format/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/ptc-script-format/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

公開されている `@deepseek-ai/dsh-ptc-runtime` サービスは `PtcRuntime.programFormat` を `'function-body' | 'repl'` から `'script'` を含むよう広げる。古い union を網羅的にマッチする外部 consumer は新しい値を処理しなければならない。`retainsState` は provider が `stateOwner` のメモリを保持するかどうかを報告する。その base 実装は `false` を返すため、カスタムの retained provider はそれを override しなければならない。

Node は `per-call` をデフォルトとして維持する。オプトインの `executionMode: vm-script` は Script 文とトップレベル await を受け付け、`scriptPersistence` が cross-call memory を独立して選択する。既存の Node function-body と REPL の構成は引き続き有効である。

## 移行

1. union リテラルをコピーする代わりに、`@deepseek-ai/dsh-ptc-runtime` からエクスポートされる `PtcProgramFormat` を import する。カスタム tool または workflow adapter の `programFormat` switch を更新する。`'script'` は文または最終式で処理し、トップレベル `return` を使わない。[Node 構文リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/ptc-runtime/ptc-runtime-node/README.md#use-this-package) を参照。
2. `programFormat === 'repl'` からメモリ保持を推論するチェックを `runtime.retainsState` に置き換える。カスタム `PtcRuntime` provider が owner スコープのメモリを保持する場合は、`retainsState` を override して `true` を返し、`release(owner)` の実装を続ける。
3. 新しいパッケージ宣言に対して外部 consumer を再ビルドする。サポートされる各形式がコンパイルされ、persistence 制御が REPL と Script の両方の provider で `retainsState` に従うことを確認する。
