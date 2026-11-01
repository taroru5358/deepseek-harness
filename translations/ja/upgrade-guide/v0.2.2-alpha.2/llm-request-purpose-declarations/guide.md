---
kind: upgrade-guide
description: "補助的な LLM リクエストの purpose は、共有の固定 union ではなく producer 所有の型宣言を必要とする。"
---

# 補助的な LLM リクエストの purpose を宣言する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/llm-request-purpose-declarations/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/llm-request-purpose-declarations/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.2-alpha.2/llm-request-purpose-declarations/guide.md](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/llm-request-purpose-declarations/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`GenerateOptions.purpose` は、`@deepseek-ai/dsh-llm/types` がエクスポートする宣言マージ可能な `LlmRequestPurposeMap` から派生する。共有パッケージは `compaction` や `session-title` を宣言しなくなった。補助リクエストを構築する plugin は、producer の宣言をロードするか、独自の purpose を宣言しなければならない。既存の runtime purpose 文字列はその意味を維持する。

## 移行

1. 直接の compaction リクエストには `@deepseek-ai/dsh-compaction-basic` を、title リクエストには `@deepseek-ai/dsh-session-title-llm` を import する。宣言だけが必要な場合は type-only import を使う。
2. plugin 所有の purpose については、plugin の公開 type エントリから `LlmRequestPurposeMap` を拡張する。purpose キーを `true` にマップする:

   ```ts
   import type {} from '@deepseek-ai/dsh-llm/types'

   declare module '@deepseek-ai/dsh-llm/types' {
     interface LlmRequestPurposeMap {
       'my-plugin-purpose': true
     }
   }
   ```

3. 新しいパッケージ宣言に対して plugin をコンパイルする。そのリクエストが宣言された purpose を受け入れ、宣言されていないキーを拒否することを確認する。
