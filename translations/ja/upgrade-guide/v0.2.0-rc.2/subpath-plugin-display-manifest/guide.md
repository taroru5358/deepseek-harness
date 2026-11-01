---
kind: upgrade-guide
description: "subpath plugin はエクスポートされた <subpath>/package.json から表示テキストやアイコンを読まなくなり、locale meta と <subpath>/icon を使う。"
---

# subpath plugin は `<subpath>/package.json` の代わりに `<subpath>/icon` を読む

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.0-rc.2/subpath-plugin-display-manifest/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.0-rc.2/subpath-plugin-display-manifest/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.0-rc.2/subpath-plugin-display-manifest/guide.md](../../../../../docs/upgrade-guide/v0.2.0-rc.2/subpath-plugin-display-manifest/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

v0.2.0-rc.2 では、Plugin Manager と Settings は、パッケージがエクスポートしている場合、`my-plugins/search` のような subpath plugin を `my-plugins/search/package.json` を通じて読んでいた。`name` と `description` が欠落した locale フィールドを埋め、`icon` が行の画像を提供した。subpath はパッケージではないため、次のリリースではそのために `package.json` を読まない。タイトルと説明は `my-plugins/search/locale/*.json` からのみ取得し、画像はエクスポートされた `my-plugins/search/icon` リソースから取得する。パッケージルートの plugin は `package.json` のテキストと `icon` を維持する。`icon` が省略された場合、エクスポートされた `./icon` が画像を提供する。subpath の `package.json` ファイルをエクスポートするパッケージの作者が影響を受ける。

## 移行

1. subpath manifest の `name` と `description` を subpath locale ファイルの `meta.title` と `meta.description` に移し、それらをエクスポートする。たとえば `"./search/locale/*.json": "./locale/search/*.json"`。
2. subpath manifest の `icon` を icon export に置き換え、その target を公開する:

   ```json
   {
     "exports": { "./search/icon": "./assets/search.svg" },
     "files": ["assets/search.svg"]
   }
   ```

3. `./search/package.json` export とそのファイルを削除する。
4. DSH チェックアウトで `pnpm run verify-package-meta` を実行するか、Plugin Manager を開き、subpath 行にタイトル、説明、画像が表示されることを確認する。
