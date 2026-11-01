---
description: "dsh-v0.1.3-alpha.2 における回顧的な Session 永続化型と隣接リリースの変更。"
kind: persistence-release
---

# 永続化リリース: dsh-v0.1.3-alpha.2

日本語 | [English](../../../../docs/persistence-changes/releases/dsh-v0.1.3-alpha.2.md) | [中文](../../../../docs/persistence-changes/releases/dsh-v0.1.3-alpha.2.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/releases/dsh-v0.1.3-alpha.2.md](../../../../docs/persistence-changes/releases/dsh-v0.1.3-alpha.2.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

既存の永続化ルートのダイジェストを変えずに、feedback/message-put と feedback/message-delete が追加される。ライター形式は 2 のままである。

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
| ソース・タグ | `dsh-v0.1.3-alpha.2` |
| ソース日付 | 2026-09-07T11:45:35.000Z |
| リリース記録 | リリース・オブジェクトあり。 |
| 前回のリリース | [dsh-v0.1.3-alpha.1](dsh-v0.1.3-alpha.1.md) |
| Session ライター・バージョン | 2 |
| 再構成されたインベントリー | <!-- persistence-release-inventory:start -->56 ルート / 435 型<!-- persistence-release-inventory:end --> |
| このスナップショット | [dsh-v0.1.3-alpha.2.schema.json](../../../../docs/persistence-changes/releases/dsh-v0.1.3-alpha.2.schema.json) |

このタグにおけるライター・バージョン定数のソース・エビデンス:

- `packages/core/session/src/types.ts`: `export const SESSION_FORMAT_VERSION = 2`

<a id="declaration"></a>
## 宣言

```yaml persistence-release
schemaVersion: 1
tag: dsh-v0.1.3-alpha.2
previous: dsh-v0.1.3-alpha.1
sessionFormatVersion: 2
changes:
  - root: event:feedback/message-delete
    before: null
    after: 3ee93b06f3a125850337602bcdf155d2538c43a5c944ec55b1b3c365152d6796
  - root: event:feedback/message-put
    before: null
    after: 3b04fde0dc763cf84fbde7b6611b3194dd56d95d0c0bf0204311640468d586e1
```

<a id="changes"></a>
## 構造上の変更

<!-- persistence-release-changes:start -->

2 個の変更されたルートと 2 個の構造上の差分を検出した。以下のレビュー要件は比較のためだけに現在のルールで計算したものであり、歴史的な準拠、移行の正しさ、ランタイム互換性を主張するものではない。

| パス | 変更 | レビュー要件 |
|---|---|---|
| `event:feedback/message-delete` | `root-added` | `not-required` |
| `event:feedback/message-put` | `root-added` | `not-required` |

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
