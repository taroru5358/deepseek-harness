---
kind: upgrade-guide
description: "WebServer の index 描画はページが代弁する Peer を取り、`webserver/index-inject` event はその Peer を listener に渡す。"
---
# WebServer の index 描画は Peer を取る

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/webserver-index-peer/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/webserver-index-peer/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/webserver-index-peer/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/webserver-index-peer/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-host-webserver` の `ctx.webServer` にある `renderIndex(html)` と `collectIndexInjections()` は Peer を取らず、すべての index が owner の access snapshot を運んでいた。現在は `renderIndex(html, peer)` と `collectIndexInjections(peer)` になり、`webserver/index-inject` event は同じ Peer を2番目の引数として渡すため、index 内の remote-access snapshot はその Peer のものになる。どちらかのメソッドを呼ぶコードはコンパイル時に壊れ、テーブルのみを読む listener は引き続き動作する。

## 移行

1. すべての `ctx.webServer.renderIndex(html)` と `ctx.webServer.collectIndexInjections()` 呼び出しに、ページが代弁する Peer を渡す。owner のブラウザに配信されるページは `ctx.connection.browserPeer` を渡す。
2. テストで `webserver/index-inject` を自身で emit する場合は、テーブルの後に Peer を渡す: `ctx.emit('webserver/index-inject', table, peer)`。
3. TypeScript ビルドを実行し、どちらのメソッドの呼び出しも引数欠落を報告しないことを確認する。
