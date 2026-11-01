---
kind: upgrade-guide
description: "plugin の互換性チェックは DSH の peerDependencies より先に engines.dsh を読み、拒否は peer ではなく要件を報告する。"
---

# plugin の互換性は `engines.dsh` を最初に読む

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.3/plugin-engines-dsh/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.3/plugin-engines-dsh/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.3/plugin-engines-dsh/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.3/plugin-engines-dsh/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

v0.2.1-alpha.3 では、インストールと profile 起動は plugin または bundle を `@deepseek-ai/dsh` と `@deepseek-ai/dsh-*` の `peerDependencies` のみで判定し、`engines.dsh` を無視していた。次のリリースでは、自身の `engines.dsh` range が単独で決定し、それらの peer は無視される。peer ルールは `engines.dsh` がない場合にのみ適用される。`workspace:` 値は peer でのみ有効で、`engines.dsh` では互換性がない。オブジェクトでない `engines` や文字列でない `engines.dsh` は、互換性のために manifest を読めなくし、パッケージを拒否する。パッケージが `engines.dsh` を宣言している、または `engines` をオブジェクト以外で宣言している plugin 作者が影響を受ける。

plugin manager が client に返す `incompatible-version` refusal レコード、`IncompatiblePlugin`、app-boot の `PluginCompatibility` 結果は `peers` を `requirements` に改名する。キーは engines ルールでは `engines.dsh`、フォールバックでは peer パッケージ名である。これらのレコードを読むコードが影響を受ける。

## 移行

1. パッケージが `engines.dsh` を宣言している場合、その range がパッケージのサポートするすべての DSH リリースをカバーするようにする。DSH の peer range はそれを広げも狭めもしなくなる。peer ベースのチェックを維持するには `engines.dsh` を削除する。
2. `engines.dsh` の `workspace:` 値をセマンティックバージョン range に置き換える。
3. `IncompatiblePlugin` と `PluginCompatibility` から `peers` の代わりに `requirements` を読む。
4. `dsh plugin --profile <profile> add <package@version>` または plugin manager でパッケージをインストールして確認する。不一致は refusal で `engines.dsh` または peer パッケージを名指しする。
