---
kind: upgrade-guide
description: "カスタムの TerminalBlock consumer は、はみ出すコマンドラインにローカライズされたアクセシブル名を提供しなければならない。"
---

# terminal コマンドのラベル

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/terminal-command-labels/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/terminal-command-labels/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/terminal-command-labels/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/terminal-command-labels/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-client-ui-primitives` の `TerminalBlockLabels` は `commandLine(line: number): string` formatter を必要とする。以前は、呼び出し元はコマンドラインにアクセシブル名を提供していなかった。水平方向にはみ出す各行は、キーボードフォーカス可能なグループにこの名前を使うようになった。収まる行（折り返された行を含む）は Tab ストップを追加しない。これは `TerminalBlock` を描画するカスタム UI plugin に影響する。

## 移行

1. `TerminalBlock` に渡す各 labels オブジェクトに `commandLine` を追加する。引数は1から始まる作成時の行番号である。
2. plugin の locale 辞書を通じて、「Command line 1」のようなローカライズされた名前を返す。元のコマンドテキストはこの名前とは別に保つ。
3. plugin を再ビルドする。長いコマンドがキーボードフォーカスを受け取り、矢印キーでスクロールできること、短いまたは折り返されたコマンドが Tab ストップを追加しないことを確認する。
