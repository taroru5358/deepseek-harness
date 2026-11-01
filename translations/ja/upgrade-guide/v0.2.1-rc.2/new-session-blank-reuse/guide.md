---
kind: upgrade-guide
description: "New Session は、呼び出しごとに新しい Agent を作成する代わりに、Workspace 内の blank Session を再び再利用する。"
---

# New Session は blank Session に戻る

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/new-session-blank-reuse/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/new-session-blank-reuse/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/new-session-blank-reuse/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/new-session-blank-reuse/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

0.2.1-rc.2 では、Web と Desktop の New Session、およびオプションなしの `uiWorkspace.startSession(workspaceId)` が、呼び出しごとに新しい Session と Agent を作成していた（[以前のガイド](../../v0.2.1-alpha.1/new-session-generations/guide.md)）。残された blank Session は Session tree から隠されるため、未送信の draft に到達できなくなった。

New Session は現在、Workspace 内の最初の未開始・未アーカイブの blank Session を再利用し、Host がそれをデフォルト preset に切り替える。存在しない場合にのみ Session を作成する。オプション付きの `startSession`、`openWorkspace`、`connectWorkspace` も同じ規則に従う。Agent preset を指定するリクエストは、一覧上の preset が異なる blank をスキップするため、Creator エントリのような preset フローが別の preset の draft を引き継がない。

Agent がまだロードされている再利用された blank は、作成時の preset 構成を保持する。New Session は現在の preset 定義を保証しなくなった。

## 移行

1. preset 定義を変更した後は、変更後に Agent が構成された Session で会話を開始する。たとえば New Session を使う前に Host を再起動する。再起動後に resume された blank は現在の定義で構成される。
2. 別の preset を列挙する blank を引き継いではならない Client 統合は、`startSession`、`openWorkspace`、`connectWorkspace` に `{ agentPreset }` を渡す。
3. 確認するには、New Session の後に draft を入力し、別の Session を開き、再度 New Session を使う。同じ blank が draft とともに開く。
