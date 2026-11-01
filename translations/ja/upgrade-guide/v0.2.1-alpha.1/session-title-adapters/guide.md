---
kind: upgrade-guide
description: "カスタム title route は登録済み adapter と、昇順の reasoning effort を必要とする。"
---

# カスタム title route を準備する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/session-title-adapters/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/session-title-adapters/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/session-title-adapters/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/session-title-adapters/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

モデルベースの session title は `ctx.llm.prepareCall()` を通じてリクエストを準備する。title route を `llm/stream` middleware のみで提供する構成も、adapter を登録しなければならない。カスタム adapter は、任意の表示順ではなく、選択可能な reasoning effort を最小から最大の順に列挙しなければならない。title policy は設定関数を通じて最初のエントリを選択する。既存の concrete な `prepareCall()` 入力は引き続き有効である。

## 移行

1. `ctx.llm.registerAdapter()` で title route を登録する。replay 構成は既存の `llm-replay` plugin の `providers` リストを設定できる。
2. adapter のモデルメタデータで、`reasoning.efforts` を選択可能な最小の effort から最大へ並べる。replay model の `reasoningEfforts` にも同じ順序を適用する。設定されたデフォルトはリスト順と独立に保つ。route に選択可能な制御がない場合は reasoning メタデータを省略する。
3. title を生成し、`session/title-llm-request` が選択された effort を記録し、provider title が保存されることを確認する。そのオプションフィールドのない既存の Session レコードは引き続き読める。
