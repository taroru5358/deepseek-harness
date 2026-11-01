---
kind: upgrade-guide
description: "dsh plugin は転送される pnpm 引数の前でのみ --profile を読み、独自の help を出力する。"
---

# `dsh plugin` のオプション配置

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/plugin-option-placement/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/plugin-option-placement/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/plugin-option-placement/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/plugin-option-placement/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`dsh plugin` は、転送される pnpm 引数の前でのみ `--profile` を読む。pnpm 引数は、認識しない最初のトークンから始まる。また、独自の help を持つ。以前はコマンドラインのどこでも `--profile <name>` を受け付け、独自の help がなく、任意の位置から `-h` と `--help` を pnpm に転送していた。

転送される引数が pnpm コマンドで始まる場合、そのコマンド以降のすべてが、`-h`、`--help`、`--profile` を含めてそのまま pnpm に届く。それ以外の位置、たとえば `--filter <selector>` のような先頭の pnpm オプションの後では、`-h` または `--help` が `dsh plugin` の help を出力して exit 0 する。`--profile` を pnpm コマンドの後や先頭の pnpm オプションの後に置くスクリプトは `required option '--profile <name>' not specified` で失敗し、`dsh plugin --profile <name> --help` は pnpm の help を出力しなくなった。

## 移行

1. `--profile <name>` をすべての pnpm 引数の前に移す。`dsh plugin add <package> --profile <name>` を `dsh plugin --profile <name> add <package>` に置き換える。
2. pnpm の help を求める場合は、そのコマンドの後に要求し、pnpm オプションもその後に置く。例: `dsh plugin --profile <name> add --help`。pnpm の一般的な help には `help` を転送する: `dsh plugin --profile <name> help`。
3. `dsh plugin --help` が `Usage: dsh plugin [options] [args...]` を出力し、手順1と2のコマンドが pnpm に届くことを確認する。
