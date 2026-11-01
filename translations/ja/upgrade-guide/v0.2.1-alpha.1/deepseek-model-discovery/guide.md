---
kind: upgrade-guide
description: "DeepSeek API-key のモデル discovery は設定済みの credential を必要とする。"
---

# DeepSeek API-key のモデル可視性

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/deepseek-model-discovery/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/deepseek-model-discovery/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/deepseek-model-discovery/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/deepseek-model-discovery/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`deepseek-official` route は以前、API キーなしで設定済みモデルを列挙していた。現在は credential がない場合に空のカタログを返すため、Desktop と Web は DeepSeek グループを隠す。モデル discovery は不正な credential と credential lookup の失敗を報告する。カタログにない保存済みまたはデフォルトの選択は、保存された選択を変えずに「Select model」を表示する。独立した DeepSeek account route は引き続き account login を使う。

## 移行

1. API-key route を使うには、Settings → Models でそのキーを設定するか、起動環境を通じて `apiKeyEnv`（デフォルト `DEEPSEEK_API_KEY`）が名指しする参照を提供する。
2. モデルセレクターを再度開き、DeepSeek グループが表示されることを確認する。カスタムの discovery consumer は、credential が設定される前の空のカタログを処理しなければならない。モデルの列挙は、リモート API がキーを受け入れることを検証しない。
