---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-10-13-deepseek-search-thinking

日本語 | [English](../../../docs/persistence-changes/2026-10-13-deepseek-search-thinking.md) | [中文](../../../docs/persistence-changes/2026-10-13-deepseek-search-thinking.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-10-13-deepseek-search-thinking.md](../../../docs/persistence-changes/2026-10-13-deepseek-search-thinking.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

記録される DeepSeek 検索リクエスト本体にオプショナルの `thinking` メンバーを追加し、補助的な検索ターンが thinking を無効にすることを記録する。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-10-13-deepseek-search-thinking
baseline: false
changes:
  - root: "event:web/deepseek-search-llm-request"
    previous: "2026-09-11-initial"
    after: "0c556c9b9f2a96238a6b9bbb99e748206cbff2ed6203041c9571f04ae5802f52"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

既存のレコードは有効なままである。欠落は、リクエストがこのフィールドを省略し、DeepSeek がそのデフォルトを適用したことを意味する。このイベントはログ専用であり、どのリーダーも振る舞いのために本体をリプレイしたり解釈したりしない。プロバイダーは新しいリクエストごとにこのメンバーを書き込む。

<a id="verification"></a>
## 検証

pnpm exec vitest run packages/web/web-search-deepseek/tests/deepseek.spec.ts: 59 件のテストに合格した。

<a id="dev-note"></a>
## 開発メモ

なし。
