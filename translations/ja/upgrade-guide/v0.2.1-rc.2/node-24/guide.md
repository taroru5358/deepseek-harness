---
kind: upgrade-guide
description: "DSH は Node.js 24.9 以降を必要とし、Node.js 22 をサポートしなくなった。"
---
# Node.js 24 が必要

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/node-24/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/node-24/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/node-24/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/node-24/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

DSH は Node.js 24.9 以降を必要とする。以前のリリースは Node.js 22.19 以降および 22.x ラインの新しいリリースもサポートしていた。この変更は npm インストール、ソースチェックアウト、Python SDK の明示的な `DSH_RUNTIME_MODE=node` 開発キャリアに影響する。パッケージ済み desktop アプリケーションと Python SDK のバンドル実行ファイルは独自の runtime を提供する。

## 移行

1. DSH の起動に使う Node.js インストールを 24.9 以降にアップグレードする。開発環境、コンテナ、CI job の Node バージョン pin を更新する。
2. 同じ環境で `node --version` を実行し、24.9 以降を報告することを確認する。
3. ソースチェックアウトでは `pnpm install` を実行する。通常使う profile で DSH を起動し、正常に起動することを確認する。
