---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-09-20-unknown-child-catalog

日本語 | [English](../../../docs/persistence-changes/2026-09-20-unknown-child-catalog.md) | [中文](../../../docs/persistence-changes/2026-09-20-unknown-child-catalog.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-09-20-unknown-child-catalog.md](../../../docs/persistence-changes/2026-09-20-unknown-child-catalog.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

読めない履歴上の子を、unknown モードで subagent/catalog に保持する。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-09-20-unknown-child-catalog
baseline: false
changes:
  - root: "event:subagent/catalog"
    previous: "2026-09-11-initial"
    after: "3abae7324356f155cb42450c00b806d134ec93bd6439d2063b8d724162d58604"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

カタログのペイロード v0 は変わらない。ペイロード v1 は unknown モードを追加し、現在のリーダーは v0 と v1 を受け入れる。完全なファクトは引き続き v0 を使い、移行は未知の子のために v1 を出力する。既存のログに書き換えは不要で、Session ヘッダーは V4 のままである。古いリーダーは v1 を拒否する。分類器がより高いイベント・ペイロード・バージョンを許すのは、古いペイロードの選択肢がすべて変わらない場合だけである。same-version での拡張と古いリーダーの削除は引き続き破壊的である。

<a id="verification"></a>
## 検証

移行、復元、プロジェクション、Web のリグレッションが、unknown のメンバーシップと子ローカルのエラーを保持する。分類器のテストは、前身を保持したより高いペイロード・バージョンを受け入れ、same-version の追加、不正なバージョン、変更または削除された古い選択肢、ヘッダーやサーフェスの変更を拒否する。

<a id="dev-note"></a>
## 開発メモ

なし。
