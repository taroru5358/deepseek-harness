---
kind: upgrade-guide
description: "@deepseek-ai/dsh-plugin-manager/operations の viewProfilePackage は、出力する manifest フィールドを必須とする。"
---

# `viewProfilePackage` は出力するフィールドを受け取る

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/plugin-manager-view-fields/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/plugin-manager-view-fields/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.2/plugin-manager-view-fields/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/plugin-manager-view-fields/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-plugin-manager/operations` の `viewProfilePackage(dir, spec, options)` は以前、固定のフィールドリスト `name`、`version`、`description`、`dsh` を出力していた。`PackageViewOptions` は現在 `fields` を必須とする。これは `pnpm view` が出力する manifest または registry フィールドであり、同じ lookup でバージョンリストとバージョンごとの peer を読めるようにする。`viewProfilePackage` を直接呼ぶコードは `fields` なしでは型チェックを通らなくなり、それを省略した JavaScript 呼び出し元はフィールドなしで `pnpm view` を実行する。

## 移行

1. 呼び出しが以前出力していたフィールドを渡す:

   ```ts
   import { viewProfilePackage } from '@deepseek-ai/dsh-plugin-manager/operations'

   declare const dir: string, spec: string, timeoutMs: number
   await viewProfilePackage(dir, spec, { fields: ['name', 'version', 'description', 'dsh'], timeoutMs })
   ```

2. type checker を実行する。`fields` プロパティの欠落が各呼び出しで報告される。
