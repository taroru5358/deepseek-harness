---
kind: upgrade-guide
description: "Reasoning Body Slot の text は表示用に準備された content を含み、元の出力は rawText に含まれる。"
---

# Reasoning Body Slot のテキストフィールド

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.4/reasoning-body-props/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.4/reasoning-body-props/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.4/reasoning-body-props/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.4/reasoning-body-props/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`conversation.chat.reasoning.body` への寄与は、新しい `rawText` フィールドでモデル出力を受け取る。既存の `text` フィールドは、元の出力ではなく Chat が準備した表示テキストを提供する。`running` は変わらない。これはカスタムの reasoning renderer と翻訳 wrapper に影響する。

`conversation.chat.reasoning.content` Factory は引き続き `text`、`running`、任意の `labels` を受け付ける。呼び出し元が選択した Markdown を描画する。[Chat README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-chat/README.md#grouped-rendering) がこの Slot を説明している。

## 移行

1. 通常の描画、翻訳リクエスト、未翻訳ビューでは `text` を読み続ける。完全なモデル出力を必要とする拡張は `rawText` を読むよう更新する。
2. 選択した `text` またはその翻訳を、Chat の表示処理を繰り返さずに Content Factory に渡す。
3. 拡張を再ビルドし、サポートされる各モデル形式でストリーム中と確定後の描画、翻訳、未翻訳ビューを確認する。
