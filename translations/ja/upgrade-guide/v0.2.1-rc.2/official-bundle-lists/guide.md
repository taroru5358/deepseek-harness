---
kind: upgrade-guide
description: "bundle リストと profile テンプレートは @deepseek-ai/dsh-app-boot から @deepseek-ai/dsh-official-bundles へ移る。"
---

# bundle リストを `@deepseek-ai/dsh-official-bundles` から import する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/official-bundle-lists/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/official-bundle-lists/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/official-bundle-lists/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/official-bundle-lists/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-app-boot` は `PROFILE_TEMPLATES`、`ProfileTemplate`、`DEFAULT_PROFILE_BUNDLES`、`OPTIONAL_BUNDLES`、`ON_DEMAND_BUNDLES`、`OFFICIAL_ON_DEMAND_CATALOG`、`OfficialBundleCatalogEntry` をエクスポートしなくなった。新しい公開パッケージ `@deepseek-ai/dsh-official-bundles` が、値と型を変えずにそれらをエクスポートする。これらの名前を `@deepseek-ai/dsh-app-boot` から import するコードは、コンパイルまたはロードに失敗する。`dsh` ランチャー、profile、bundle 選択は従来どおり動作する。

## 移行

1. これらの名前を import する各パッケージの `dependencies` に `@deepseek-ai/dsh-official-bundles` を追加する。
2. それらの import を `@deepseek-ai/dsh-app-boot` から `@deepseek-ai/dsh-official-bundles` に変更する。例: `import { PROFILE_TEMPLATES } from '@deepseek-ai/dsh-official-bundles'`。`initProfile` や `loadProfile` など他のすべての名前は引き続き `@deepseek-ai/dsh-app-boot` から import する。
3. 型チェックまたはビルドを実行する。`@deepseek-ai/dsh-app-boot` から移動した名前の import が残っていなければ成功する。各リストについては [パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/util/official-bundles/README.md) を参照。
