---
kind: upgrade-guide
description: "credential を名指しする pi-ai route は、credential が設定されるまでモデルを隠す。"
---

# credential なしの pi-ai モデル可視性

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/pi-ai-model-discovery/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/pi-ai-model-discovery/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.2/pi-ai-model-discovery/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/pi-ai-model-discovery/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`apiKeyEnv` を通じて credential を名指しする pi-ai route は以前、その credential がない間も設定済みモデルを列挙し、リクエストは `MISSING_CREDENTIAL` で失敗していた。モデル discovery は、credential が解決するまでそのような route に空のカタログを返すため、Desktop と Web はそのグループを隠す。不正なキーなど他の credential エラーは引き続きモデル discovery を失敗させる。provider のネイティブ認証を使う、または credential を名指ししない route は、設定済みカタログを保持する。カタログにない保存済みまたはデフォルトの選択は、保存された選択を変えずに「Select model」を表示する。

## 移行

1. そのような route を使うには、Settings → Models でそのキーを設定するか、起動環境を通じて `apiKeyEnv` が名指しする参照を提供する。
2. モデルセレクターを再度開き、その route のグループが表示されることを確認する。カスタムのモデル discovery consumer は、credential が設定される前の空のカタログを処理しなければならない。カタログが列挙されていても、リモート API がキーを受け入れることを意味しない。
