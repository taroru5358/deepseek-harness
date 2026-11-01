---
kind: upgrade-guide
description: "Client plugin は接頭辞付きの context menu 型と hook を使い、キー付きのアイコンコンポーネントを登録しなければならない。"
---

# context menu の型とアイコン

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/context-menu-target/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/context-menu-target/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.2-alpha.2/context-menu-target/guide.md](../../../../../docs/upgrade-guide/v0.2.2-alpha.2/context-menu-target/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-client-ui-context-menu/client` の export `MenuTarget` は `ContextMenuTarget` に改名され、`ContextMenuTargetMap` と一致する。旧名に互換 alias はない。Target フィールドと runtime の動作は変わらない。workspace と sidebar の injection は `menuTarget` / `useMenuTarget` を `contextMenuTarget` / `useContextMenuTarget` に改名する。アクションアイコンは React node の代わりに `{ key, src? }` を受け付け、キー付きの `contextMenu.icon` Slot を通じて描画する。

## 移行

1. `MenuTarget` の import と型参照を `ContextMenuTarget` に置き換える。
2. inject された `menuTarget` source と `useMenuTarget` consumer を `contextMenuTarget` と `useContextMenuTarget` に改名する。
3. アクションアイコン要素を `{ key: 'my-plugin.action' }` に置き換える。必要に応じて捕捉した画像 URL を `src` に設定する。そのキーの下で `ctx.slots.inject('contextMenu.icon', ...)` と `ctx.slots.register()` を通じてアイコンコンポーネントを登録する。[menu パッケージ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-context-menu/README.md#use-this-package) を参照。
4. カスタム Client plugin を再ビルドし、型チェックが通ることを確認し、メニューを開いてアイコン描画と plugin アンロード時の cleanup を確認する。
