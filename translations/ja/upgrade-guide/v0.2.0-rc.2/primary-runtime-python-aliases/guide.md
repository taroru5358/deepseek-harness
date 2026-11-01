---
kind: upgrade-guide
description: "カスタムの primary-runtime ペイロードは再配置可能な Python インタープリター alias を必要とする。"
---

# カスタム Python runtime の alias は sibling interpreter を指さなければならない

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.0-rc.2/primary-runtime-python-aliases/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.0-rc.2/primary-runtime-python-aliases/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.0-rc.2/primary-runtime-python-aliases/guide.md](../../../../../docs/upgrade-guide/v0.2.0-rc.2/primary-runtime-python-aliases/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

macOS と Linux では、`DSH_PRIMARY_RUNTIME` は以前、ペイロード外や他のリンクを通じて解決されるインタープリターリンクを受け付けていた。ペイロード検証は現在、リンクされた各 `dependencies/python/bin/python` または `python3` が、`runtime.json` の Python major/minor バージョンにちなんで名付けられた通常の sibling interpreter（たとえば `python3.12`）を直接指すことを要求する。Python wheel で配布されるものを含め、通常のコマンドファイルは引き続きサポートされる。Windows は変わらない。

## 移行

1. カスタムペイロードでは、バージョン付きインタープリターを `dependencies/python/bin/` に置き、`python` と `python3` をそのファイル名への相対リンクにする。[runtime 準備の手順](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/skill/tool-workspace-dependencies/README.md) が必要な target と output 引数を列挙している。
2. 準備済みリソースをコピーする際は相対リンクを保持する。Node の `fs.cp` では `verbatimSymlinks: true` を使う。wheel builder はコマンドファイルを実体化しなければならない。pip はリンクを復元しない。ネイティブビルドの smoke は、準備済みリソースツリーが両方の alias を保持することを要求する。
3. `readlink` が両方の alias について sibling のファイル名のみを報告することを確認し、ペイロードをビルドディレクトリから移動して `DSH_PRIMARY_RUNTIME` を新しいディレクトリに向け、`load_workspace_dependencies` と返された Python コマンドが成功することを確認する。
