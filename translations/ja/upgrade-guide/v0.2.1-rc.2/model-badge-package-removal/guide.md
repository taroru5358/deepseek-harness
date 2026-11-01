---
kind: upgrade-guide
description: "固定の 1017 モデル badge パッケージは削除され、モデル badge は DeepSeek のモデル一覧から取得される。"
---
# `ui-deepseek-model-label-1017` パッケージの削除

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/model-badge-package-removal/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/model-badge-package-removal/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/model-badge-package-removal/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/model-badge-package-removal/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-client-ui-deepseek-model-label-1017` と、Web bundle の `ui-deepseek-model-label-1017` 行が削除され、パッケージは今後のリリースを受け取らない。composer のモデルセレクターは、DeepSeek の `GET /models` listing がモデルに対して宣言する badge を表示する。宣言された badge がない場合、`DeepSeek-V4.1-Flash` は `1017` を表示しない。削除された行を無効化または編集する profile パッチはどのエントリにも一致しないため、boot はエントリが見つからないと警告し、パッチをスキップする。

## 移行

1. `id` が `ui-deepseek-model-label-1017` のパッチを、その `disabled` や他のフィールドとともに、`$DSH_HOME/profiles/<profile>/cordis.patch.yml` と `--patch` で渡すすべての overlay から削除する。
2. それを列挙しているパッケージの依存関係から `@deepseek-ai/dsh-client-ui-deepseek-model-label-1017` を削除する。
3. モデル名や badge の描画方法を変更するには、`conversation.input.model.label` slot に寄与する。その owner props にはカタログの `model.badge` が含まれる（[label 拡張](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-model-selection/README.md#use-this-package)）。
4. DSH を再起動し、boot が `ui-deepseek-model-label-1017` エントリの欠落を報告しないことを確認する。
