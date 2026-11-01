---
kind: upgrade-guide
description: "Claude Code と Codex の bundle はパッケージと行の識別子を維持したまま、グローバルな委譲 tool を追加する。"
---

# ネイティブ subagent bundle はグローバル tool を含む

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/native-subagent-bundle-tools/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/native-subagent-bundle-tools/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/native-subagent-bundle-tools/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/native-subagent-bundle-tools/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-subagent-claude-code` と `@deepseek-ai/dsh-subagent-codex` は、パッケージ名、bundle 選択、provider 行 ID、設定、ネイティブ認証を維持する。選択された bundle は、その委譲 tool を Host グローバル tool としても登録し、minimal を含むすべての preset から見えるようになった。以前は bundle は provider だけを登録していた。ライブな Agent は bundle が有効になった後の次のリクエストから tool を見え、bundle が無効になると失う。

提供される Host 行は `tool-subagent-claude-code` と `tool-subagent-codex` である。後続のユーザーレイヤーは行 id でそれらの設定や disabled 状態を override できる。override は選択されていない bundle を選択しない。同じ名前の独自 tool を登録する preset は、その Agent に対してグローバル tool を shadow する。

委譲は共有の managed-activation API に従う。tool は子 ID を返し、完了は親 Agent へのログ記録された通知として届く。不要になった `backgroundMode` 設定を削除する。`maxDepth: provider-managed` はこれらのネイティブ provider で引き続きサポートされる。

## 移行

1. 選択した bundle 名、provider 設定、認証を維持する。provider のみの既存パッケージは、tool を提供する bundle を受け取るために Plugins → Official で **Update** が必要である。要求されたら再起動する。新しい provider を採用するには、そのカードを有効にする。[インストール対象](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-manager/README.md#use-this-package) は公開された DSH バージョンまたはソースチェックアウトに従う。同じバージョン文字列でも、古い registry パッケージが開発リンクと等価になるわけではない。Off は選択解除のみで、Remove は別である。
2. 通常の行パッチで tool を設定する。設定 override は引き続き完全な設定を置き換える。たとえば、次のようにすると provider を登録したまま Codex tool を無効にできる:

   ```yaml
   - id: tool-subagent-codex
     disabled: true
   ```

3. 手動の Host 行が同じ tool 名を登録している場合は、その冗長な行を削除するか、bundle の行を無効にする。Session で有効な tool カタログを確認する。
4. Plugin client は必須の `BundleInfo.official` と `BundleInfo.availability` フィールドを受け取る。detail-slot の `PluginPackageRef` も同様である。`official` はプロジェクトの保守状況を識別し、`availability` は `installation`、`profile`、`missing` を報告し、`installed` は引き続き profile 依存関係の宣言を意味する。既存フィールドのみを表示する detail-slot のリーダーは変更不要である。constructor と厳密な validator は新しいフィールドを含めなければならない。任意の `installTarget` は、提供されるバージョンと registry またはローカルリンク spec を識別する。

**その他の profile。** bundle レイヤーは Host 行のみを挿入するため、headless、SDK、ACP、カスタム profile もそれらを選択できる。provider パッケージを維持しつつ bundle を選択しない profile は、[Codex](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/subagent-codex/README.md#exposing-the-tool) または [Claude Code](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/subagent-claude-code/README.md#exposing-the-tool) の例を使って同じ行を構成できる。
