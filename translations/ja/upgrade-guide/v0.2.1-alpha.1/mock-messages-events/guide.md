---
kind: upgrade-guide
description: "Messages mock stream はネイティブ SDK consumer 向けに名前付き SSE event を公開する。"
---

# Messages mock server から名前付き event を読む

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/mock-messages-events/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/mock-messages-events/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/mock-messages-events/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/mock-messages-events/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

次のリリースでは、`@deepseek-ai/dsh-llm-mock-server` が各構造化 SSE payload の横に Messages event 名を emit する。ネイティブ SDK は message と content-block event を処理するためにこれらの名前を必要とする。名前のない SSE メッセージのみを受け入れる consumer は、event 処理を更新しなければならない。script 化された payload、順序、リクエスト捕捉、チャンク数は変わらない。意図的に不正な JSON の動作は、引き続き無効なデータを直接送る。

## 移行

1. 名前付き Messages event を消費し、その `data` フィールドを parse する。ストリームを名前のない event にフィルタしない。
2. 完全なテキスト応答、tool-call 応答、テストが使う fault 動作を確認する。[mock server リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/test-support/llm-mock-server/README.md) が利用可能な script を説明している。
