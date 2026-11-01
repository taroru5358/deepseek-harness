---
kind: upgrade-guide
description: 既存の SDK profile はネイティブ desktop tool とモデル context を含む。
---

# SDK の desktop tool と context

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/sdk-desktop-context/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/sdk-desktop-context/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/sdk-desktop-context/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/sdk-desktop-context/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

既存の `sdk` profile は `present`、`ask_user_question`、schedule tool、file-reference guidance、time context を含むようになった。Web サーバーは起動しない。ネイティブ tool 実行が引き続きデフォルトで、PTC は [executor overlay](../tool-executor-plugins/guide.md) を通じて選択する。`sdk-minimal` は変わらない。

## 移行

1. `sdk` を選択し続ける。ネイティブプロトコル経由で組み込みの質問に回答するには `onUserQuestion`（TypeScript）または `on_user_question`（Python）を提供するか、in-process の `user-questions/request` 回答者を保持する。どちらもない場合、tool は `NO_PROVIDER` を報告する。
2. working directory とモデルは引き続き SDK の初期化から取得する。SDK profile は Web 固有の checkout と GUI 開発 context をマウントしない。
3. 以前のモデル可視 tool/context roster を必要とするカスタム構成は、`present`、`tool-ask-user`、`tool-schedule`、`file-reference-local`、`deliverable-file-references`、`time-context` の行を無効にできる。アップグレード前に、固定した prompt と tool schema を見直す。
