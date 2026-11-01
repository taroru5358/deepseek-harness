---
kind: upgrade-guide
description: "カスタム Client 構成は、GUI plugin が必要とする context-menu サービスと file-action サービスを提供しなければならない。"
---

# カスタム Client 構成における context menu サービス

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/context-menu-services/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/context-menu-services/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/context-menu-services/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/context-menu-services/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

GUI plugin は共有オブジェクトメニューを開くために `contextMenus` を inject するようになった。open-in-app plugin は `fileActions`、`sidebarRight`、`sidebarRightTabs`、`remote.workspaceFiles` も inject する。新しいサービスなしでこれらの GUI plugin をマウントするカスタム構成は、それらを activate できない。出荷される Web アプリケーション bundle は provider を含む。既存の Session と tab の menu Slot 名は引き続き利用可能である。

## 移行

1. カスタム構成を所有するパッケージ manifest で、同じ Harness バージョンの `@deepseek-ai/dsh-client-ui-context-menu` と `@deepseek-ai/dsh-client-ui-file-actions` への依存関係を追加する。
2. 既存の locale、slots、shortcuts、sidebar、workspace-file provider と並べて、両方のパッケージ名をカスタム Client 構成の plugin 行として追加する。デフォルトの `@deepseek-ai/dsh-web-app` bundle はすでにこれらの行を含む。
3. `ctx.menus` を `ctx.contextMenus` に、`MenuTargetMap` を `ContextMenuTargetMap` に改名する。どちらの旧名にも alias はない。menu plugin をマウントし、その `ctx.slots.provideRoot` 寄与がすべての Slot に `useContextMenu` を提供するようにする。コンポーネントは hook を1回呼び、ループ内で作成された要素を含む各要素に `getTriggerProps(target, handlers?)` を spread する。handler の構成と外部アクション登録については [menu リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-context-menu/README.md) を参照。
4. カスタム profile を再ビルドして起動する。GUI plugin が activate すること、背景の Session を右クリックしても selection が保持されること、既存の Session/tab Slot 寄与が引き続き表示されることを確認する。
5. カスタム Slot host は `sidebar.workspaces.session.menu.item` と `sidebar.right.tab.menu.item` の宣言を削除しなければならない。workspace と right-sidebar の adapter は `contextMenu.content` の下でこれらを所有するようになった。Session 行は `[open, setOpen]` hook context を保持し、tab 行はクリックされた Session の provider 内で `{ tab, dismiss }` を受け取る。これらの adapter を置き換えるカスタム host は、同じ children を宣言し、それらの owner を自身で提供しなければならない。`getTabTriggerProps` を使う Dockkit embedder は popup と dismissal を所有する。`renderTabMenuItems` は kit popup のみが使う。カスタム menu の準備中または開いている間に close-button tooltip を抑制するには、リアクティブな `tabMenuOpen` を提供する。

6. 組み込み Session menu item はデータ寄与である。レガシーな React Slot は外部エントリを Archive の前にまとめて保持する。Slot priority は組み込みアクションを shadow しなくなった。新しいアクションには `ctx.contextMenus.register` を使い、任意の `commandId` で既存コマンドの現在の binding を表示する。捕捉した target での実行は `run` に保つ。コマンド dispatch は追加されない。
7. menu 寄与は `icon?: ReactNode` を使う。`symbol` を glyph 要素に置き換え、icon URL を `<img>` でラップする。実際の各 Slot エントリで `useContextMenu` を取得する。プレーンな children は `getTriggerProps` またはバインドされた `contextMenuProps` を受け取る。ToolRow は `fileContextMenuProps` と `urlContextMenuProps` を使う。Chat flow/node owner から menu フィールドを削除する。assistant エントリは `forClosing()` に通常の file adapter を提供し、Markdown は `MarkdownDelegateProvider.getTriggerProps` でその target 型を適応させる。Session identity と gesture 時の path 解決はビジネス登録に保つ。`contextMenuTrigger` と `menuPoint` の import を削除する。それらの独立した utility パッケージは削除された。ui-primitives に gesture export を追加しない。
