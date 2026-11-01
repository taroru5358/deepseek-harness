---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-09-18-schedule-optional-title

日本語 | [English](../../../docs/persistence-changes/2026-09-18-schedule-optional-title.md) | [中文](../../../docs/persistence-changes/2026-09-18-schedule-optional-title.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-09-18-schedule-optional-title.md](../../../docs/persistence-changes/2026-09-18-schedule-optional-title.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

永続化された schedule/change の作成レコードの after、at、every の各バリアントで、保存される title をオプショナルにする。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-09-18-schedule-optional-title
baseline: false
changes:
  - root: "event:schedule/change"
    previous: "2026-09-11-initial"
    after: "a0a2e5c42e1c929445ecd1cd70f49be6b66441894ec72c08e8ae332821d4a3cb"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

title が存在する前に書かれたバージョン 1 の Session イベントには title メンバーがないため、それらのログは拒否されずに、title なしでデコードされ折り畳まれる。名前を要求するリーダーは、title の欠落を名前のないタスクとして扱う。Host のタスク・レコードは引き続きこのメンバーを要求する。ストレージのデコーダーはそれのない保存済みタスクを拒否し、作成、更新、スケジュール・ツールも引き続きそれを要求する。

<a id="verification"></a>
## 検証

pnpm exec vitest run packages/schedule/schedule/tests: 18 ファイル、789 件のテストに合格した。

<a id="dev-note"></a>
## 開発メモ

なし。
