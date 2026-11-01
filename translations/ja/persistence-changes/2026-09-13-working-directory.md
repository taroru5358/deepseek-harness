---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-09-13-working-directory

日本語 | [English](../../../docs/persistence-changes/2026-09-13-working-directory.md) | [中文](../../../docs/persistence-changes/2026-09-13-working-directory.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-09-13-working-directory.md](../../../docs/persistence-changes/2026-09-13-working-directory.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

Session ごとの永続的な作業ディレクトリ変更を追加する。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-09-13-working-directory
baseline: false
changes:
  - root: "event:working-directory/change"
    previous: null
    after: "124aa62078ca3311353b39cc3bf6c75da281338be6bbf672f8b361d828cddca3"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

既存の Session ヘッダーとイベントは有効なままである。新しいイベントは読み取り時に必須であり、working-directory/change を知らない古いビルドはそれを含むログを拒否する。イベントの語彙は増えるが、Session の構造的な形式は変わらない。

<a id="verification"></a>
## 検証

対象を絞った working-directory と system-prompt のスイートが 113 件のテストに合格し、ファイルごとに 100% をカバーする。キー不要のヘッドレス・リプレイが、変更、復旧、ユーザー・コンテキストの通知、安定した system prompt を検証する。

<a id="dev-note"></a>
## 開発メモ

なし。
