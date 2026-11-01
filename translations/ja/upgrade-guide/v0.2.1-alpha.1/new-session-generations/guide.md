---
kind: upgrade-guide
description: "New Session は既存の空 Session を再利用せず、新しい Agent を作成する。"
---

# New Session で新しい Agent を開始する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/new-session-generations/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/new-session-generations/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/new-session-generations/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/new-session-generations/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

Web と Desktop では、New Session は現在の preset 定義を使って新しい Session と Agent を作成する。以前は workspace 内の既存の空 Session を再利用し、以前の preset 定義が残ることがあった。

`uiWorkspace.startSession(workspaceId)` への Client 呼び出しも同じ動作に従う。`openWorkspace` または `connectWorkspace` による workspace の再接続は、引き続き適格な blank を再利用する。draft 準備オプション付きの呼び出しは、既存の再利用と content の規則を維持する。既存の Agent は preset 構成を保持し、グローバルなオプション tool は bundle の切り替え時に次のリクエストで更新される。

## 移行

1. preset 定義を変更した後は、New Session を使って現在の定義で Agent を作成する。保持された preset 構成が意図的な場合は既存の Session で続行する。
2. workspace を再接続する Client 統合は `openWorkspace` または `connectWorkspace` を使うべきである。既存の draft を準備する場合は、明示的な draft 準備オプションを保持する。
3. 新しい Agent が更新された preset 定義を使うことを確認する。永続化された Session ファイルや profile パッチを書き換える必要はない。
