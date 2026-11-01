---
kind: upgrade-guide
description: "Python runtime wheel は Office 変換とバンドルされた authoring 環境のために明示的なダウンロードを必要とする。"
---

# オプションの Python runtime リソースを明示的にダウンロードする

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/python-optional-resources/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/python-optional-resources/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/python-optional-resources/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/python-optional-resources/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

Python runtime wheel は Office sidecar と authoring interpreter・library を省略する。SDK startup はそれらをダウンロードしない。Office 操作には明示的なダウンロードが必要で、authoring 環境と conversion engine は独立して選択できる。SDK は引き続き通常の `dsh` CLI をパッケージし、システム Node.js を必要としない。

## 移行

1. Office ファイルを処理する前に `deepseek_harness_runtime.download_office()` を呼ぶ。バンドルされた authoring 環境を使う前に `deepseek_harness_runtime.download_primary_runtime()` を呼ぶ。どちらも完了済みキャッシュを再利用し、絶対パスを返す。
2. オフラインデプロイでは、対象プラットフォームで一致する runtime リリースを使ってキャッシュを準備し、そのプラットフォームのデプロイにコピーする。ダウンロードと SDK startup の前に `DSH_RESOURCE_CACHE` をその絶対ルートに設定する。[runtime リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/python/sdk-runtime/README.md) を参照。
3. SDK を起動し、必要なリソースが利用可能であることを確認する。`deepseek_harness_runtime.resolve_office_launch_args()` はダウンロードせずに Office CLI パスを確認する。既存の `DSH_PRIMARY_RUNTIME` パスは引き続きサポートされ、空の値は authoring クエリを無効にする。profile パッチは `skill-office` を独立して無効にできる。
