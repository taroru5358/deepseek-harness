---
kind: upgrade-guide
description: "`--from-default-profile` は既存のカスタム profile を拒否せず、置き換えるようになった。"
---

# `--from-default-profile` は既存の profile を置き換える

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/from-default-profile-replaces-target/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/from-default-profile-replaces-target/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.2/from-default-profile-replaces-target/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/from-default-profile-replaces-target/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`dsh --profile <name> --from-default-profile <template>` は以前、target profile が存在する場合に `profile "<name>" already exists` で失敗し、残存する target ディレクトリを拒否していた。現在は毎回、template の新しいコピーで target ディレクトリを置き換えてから起動するか、config-dump フラグ付きの場合は構成を出力する。以前のディレクトリは、その `package.json` 依存関係、インストール済み `node_modules`、`cordis.patch.yml` を含めて削除される。出荷される profile 名は、大文字小文字にかかわらず target として引き続き拒否される。

これは、スクリプトやショートカットで `dsh rescue --from-default-profile web` のようなコマンドを繰り返し、後の実行で profile の plugin やパッチが保持されることを期待している人に影響する。

## 移行

1. 既存の profile をリセットせずに起動するには、オプションを外す。`dsh rescue --from-default-profile web` ではなく `dsh rescue` を実行する。
2. リセットをまたいでローカル変更を保持するには、コマンド実行前に `$DSH_HOME/profiles/<name>/cordis.patch.yml` をコピーし、その後 `dsh plugin --profile <name> add <package>` で profile の plugin を再インストールし、パッチを復元する。
3. 移行を確認する: `dsh rescue --from-default-profile web --help` の後、`$DSH_HOME/profiles/rescue/package.json` が依存関係を列挙しない。
