---
kind: upgrade-guide
description: "JSONL persistence は namespace plugin import を使う。直接の service-class 構築と class 型注釈は persistence service を使わなければならない。"
---

# JSONL persistence をその plugin を通じてマウントする

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/jsonl-persistence-plugin/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/jsonl-persistence-plugin/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/jsonl-persistence-plugin/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/jsonl-persistence-plugin/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-session-persistence-jsonl` は、名前付きの `name`、`Config`、`apply` export を通じて関数 plugin をエクスポートし、default export を持たない。default import、直接構築、subclassing、`instanceof`、class 型注釈が影響を受ける。plugin は、この provider の削除によってトリガーされる consumer unload を通じて listener と handle を保持する。同時の root disposal は引き続き順序を持たない。`sessionPersistence` サービス API、YAML 設定、保存ファイルは変わらない。

## 移行

1. default import を `import * as JsonlSessionPersistence from '@deepseek-ai/dsh-session-persistence-jsonl'` に置き換える。既存の `ctx.plugin(JsonlSessionPersistence, config)` 呼び出しは引き続き有効である。直接構築をこのマウント呼び出しに置き換える。dynamic import では、`.default` ではなく `await import(...)` が返す module namespace を使う。consumer は `inject: ['sessionPersistence']` を宣言し、`ctx.sessionPersistence` にアクセスする。
2. サービス注釈には `@deepseek-ai/dsh-session-persistence` の `SessionPersistence` 型を使う。concrete class の `instanceof` チェックと subclass をそのサービスの consumer に置き換える。concrete な JSONL class は private である。
3. plugin の startup を await し、session を作成、flush、再度開けることを確認する。非同期 consumer cleanup は `sessionPersistence` を inject する scope に登録し、backend のアンロードがそれを待つようにする。
