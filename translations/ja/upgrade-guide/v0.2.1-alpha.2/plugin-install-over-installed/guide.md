---
kind: upgrade-guide
description: "HMR では、すでにインストールされているパッケージのインストールは already-installed で失敗し、アップグレードはパッケージを先に削除する。"
---

# HMR 下でインストール済み plugin パッケージへのインストールは拒否される

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/plugin-install-over-installed/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/plugin-install-over-installed/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.2/plugin-install-over-installed/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/plugin-install-over-installed/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

HMR を実行する profile（出荷される profile は実行する）では、`pluginManager.installBundle(spec)` は以前、profile がすでに依存しているパッケージに対して pnpm を実行し、以前のバージョンが動き続けたまま `restart-required` を報告していた。現在は、spec がインストール済みパッケージを名指しする場合、pnpm の実行前に `application: 'failed'` と `error.code: 'already-installed'` で失敗し、profile ファイルを変更しない。インストール済みパッケージとは、バージョン付きまたはバージョンなしの registry 名、ローカルディレクトリ、依存関係が保存された git または tarball spec である。同じ spec のインストールを繰り返すと、`changed: false` を報告する代わりに同じように失敗する。

これは Host API と `@deepseek-ai/dsh-api-remotes/client` の remote client を通じた `installBundle()` の呼び出し元、および agent の `plugin_manager` tool（`install_bundle`）に影響する。plugin 管理を含む bundle のように削除できないインストール済みパッケージは、HMR の実行中に上書きインストールできなくなった。公式 plugin の Web ページの Update は、パッケージを削除して新しいバージョンを自身でインストールする。HMR なしでは、インストール済みパッケージへのインストールは引き続き pnpm を実行し、`restart-required` を報告する。

## 移行

1. インストール済みパッケージの別バージョンや別の場所をインストールするには、先に `removeBundle(name)` を呼び、次に `installBundle(spec)` を呼ぶ。agent tool では `remove_bundle`、次に `install_bundle` を呼ぶ。削除は bundle の plugin を停止し、そのパッケージのマッピングを落とすため、新しいバージョンは新たにマッピングされ、再起動なしで起動する。
2. コードが `ManagementError['code']` を網羅的に switch している場合は、`already-installed` コードを処理する。
3. 移行を確認する: パッケージを再度インストールすると `already-installed` を返し、`removeBundle` 後の同じ `installBundle` 呼び出しは新しい `version` とともに `applied` を返す。
