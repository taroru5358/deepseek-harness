---
kind: upgrade-guide
description: "非 page Client はデフォルトで forwarded interaction を行わず、ブラウザ回復なしの raw stream を使う。"
---

# 非 page client の interaction 参加を選択する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/operator-connect/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/operator-connect/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.2/operator-connect/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/operator-connect/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

非 page Connection のインストールはデフォルトで `interactions: 'none'` になる。`$events` を開かず、そのモードでは `$on()` と generation-recovery helper の `$stream()` は利用できない。生成された unary と raw stream メソッドは引き続き利用できる。ブラウザ認証、既存のブラウザ Cookie、共有 Peer、回復動作は変わらず、ブラウザは再認証を必要としない。

## 移行

1. `GET /connect?token=<launch-token>&interactions=none` で launch token を交換し、リダイレクトを手動で処理する。返された API Cookie を client 所有の storage に保持し、HTTP と、返された 302 Location の lifecycle WebSocket に使う。HTTP は socket の open を待つ必要はない。
2. 受動的な streaming には、`$stream()` ではなく生成された raw stream メソッドを直接呼ぶ。lifecycle socket の閉鎖または Host の再起動後は、再試行前に新しい connect を行う。
3. 呼び出し元が forwarded interaction に意図的に応答する場合は、`interactions=answer` を要求し、同じ mode で Connection をインストールする。受動的な呼び出し元が両方の event endpoint で拒否され、応答する呼び出し元が subscribe できることを確認する。
