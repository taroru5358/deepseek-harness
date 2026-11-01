---
description: "dsh-v0.1.1-rc.1 における回顧的な Session 永続化型と隣接リリースの変更。"
kind: persistence-release
---

# 永続化リリース: dsh-v0.1.1-rc.1

日本語 | [English](../../../../docs/persistence-changes/releases/dsh-v0.1.1-rc.1.md) | [中文](../../../../docs/persistence-changes/releases/dsh-v0.1.1-rc.1.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/releases/dsh-v0.1.1-rc.1.md](../../../../docs/persistence-changes/releases/dsh-v0.1.1-rc.1.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

permission/preset が、default、selection、inferred の値を持つオプショナルの origin を得る。ライター形式は 0 のままである。

## 目次

- [リリースのエビデンス](#evidence)
- [宣言](#declaration)
- [構造上の変更](#changes)
- [検証](#verification)
- [開発メモ](#dev-note)

-----

<a id="evidence"></a>
## リリースのエビデンス

このおおよそのバックフィルは読解と形式の検証を支えるものであり、当時の互換性の承認ではない。抽出と対象範囲の制限については[アーカイブのリファレンス](index.md)を参照のこと。

| 項目 | 記録値 |
|---|---|
| ソース・タグ | `dsh-v0.1.1-rc.1` |
| ソース日付 | 2026-08-21T06:21:44.000Z |
| リリース記録 | リリース・オブジェクトあり。 |
| 前回のリリース | [dsh-v0.1.0-rc.8](dsh-v0.1.0-rc.8.md) |
| Session ライター・バージョン | 0 |
| 再構成されたインベントリー | <!-- persistence-release-inventory:start -->51 ルート / 407 型<!-- persistence-release-inventory:end --> |
| このスナップショット | [dsh-v0.1.1-rc.1.schema.json](../../../../docs/persistence-changes/releases/dsh-v0.1.1-rc.1.schema.json) |

このタグにおけるライター・バージョン定数のソース・エビデンス:

- `packages/core/session/src/types.ts`: `export const SESSION_FORMAT_VERSION = 0`

<a id="declaration"></a>
## 宣言

```yaml persistence-release
schemaVersion: 1
tag: dsh-v0.1.1-rc.1
previous: dsh-v0.1.0-rc.8
sessionFormatVersion: 0
changes:
  - root: event:permission/preset
    before: 5c45bf4c544a7211dcd8ba6ba7e5f1bc39b49e7a9df9d5cbdc8e87c22771b37b
    after: 7271e4b771406aaf06014c2269edd6cb68055bb8b8686571730813ce0ababc22
```

<a id="changes"></a>
## 構造上の変更

<!-- persistence-release-changes:start -->

1 個の変更されたルートと 1 個の構造上の差分を検出した。以下のレビュー要件は比較のためだけに現在のルールで計算したものであり、歴史的な準拠、移行の正しさ、ランタイム互換性を主張するものではない。

| パス | 変更 | レビュー要件 |
|---|---|---|
| `event:permission/preset.data.origin` | `optional-property-added` | `not-required` |

<!-- persistence-release-changes:end -->

<a id="verification"></a>
## 検証

抽出は正規グラフ、ルート・ダイジェスト、到達可能な型のダイジェストの検証に合格しており、元のオプショナルの `surfaceOp` は履歴上のサーフェス・イベントに限って許可している。ツリー内のチェックは、各タグをその前身から再構成し、before/after の値、スナップショットの網羅性、バイリンガルの機械宣言を検証する。

```sh
pnpm run verify-persistence-releases
```

<a id="dev-note"></a>
## 開発メモ

なし。
