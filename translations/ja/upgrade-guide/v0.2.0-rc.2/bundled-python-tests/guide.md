---
kind: upgrade-guide
description: "バンドルされた NumPy と pandas は上流の test suite を省略する。"
---

# NumPy と pandas の上流テストには別の環境が必要

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.0-rc.2/bundled-python-tests/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.0-rc.2/bundled-python-tests/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.0-rc.2/bundled-python-tests/guide.md](../../../../../docs/upgrade-guide/v0.2.0-rc.2/bundled-python-tests/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

Desktop と Python SDK の runtime 配布物は以前、NumPy と pandas の test ディレクトリを含んでいた。バンドル配布物は現在これらのディレクトリを省略するため、test module を import したり完全な上流スイートを実行したりするスクリプトには別のインストールが必要である。データ処理、Office authoring、`numpy.testing`、`pandas.testing`、`pandas._testing` は引き続き利用可能である。

## 移行

1. スクリプトが `numpy.*.tests` や `pandas.tests` を import する場合は、元の NumPy と pandas の配布物とその上流テスト依存関係を別の Python 環境にインストールする。その環境は、アップグレードで置き換えられる管理対象の Harness runtime ディレクトリの外に保つ。
2. 影響を受けるスクリプトや上流スイートを、その環境のインタープリターで実行する。ライブラリ機能や保持された testing helper のみを使うスクリプトは変更不要である。
3. 以前失敗していた test module の import と影響を受けるスクリプトが、別のインタープリターで成功することを確認する。バンドルされたバージョンは [runtime lock](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/primary-runtime/lock.json) に記録されている。
