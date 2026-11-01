---
description: "dsh-v0.1.5-alpha.2 における回顧的な Session 永続化型と隣接リリースの変更。"
kind: persistence-release
---

# 永続化リリース: dsh-v0.1.5-alpha.2

日本語 | [English](../../../../docs/persistence-changes/releases/dsh-v0.1.5-alpha.2.md) | [中文](../../../../docs/persistence-changes/releases/dsh-v0.1.5-alpha.2.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/releases/dsh-v0.1.5-alpha.2.md](../../../../docs/persistence-changes/releases/dsh-v0.1.5-alpha.2.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

deliverables/presented と subagent/catalog が追加される。feedback レコードはオプショナルの category を得て、feedback/record の text はオプショナルになり、ライター形式は 3 のままである。

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
| ソース・タグ | `dsh-v0.1.5-alpha.2` |
| ソース日付 | 2026-09-09T14:13:03.000Z |
| リリース記録 | リリース・オブジェクトあり。 |
| 前回のリリース | [dsh-v0.1.5-alpha.1](dsh-v0.1.5-alpha.1.md) |
| Session ライター・バージョン | 3 |
| 再構成されたインベントリー | <!-- persistence-release-inventory:start -->59 ルート / 462 型<!-- persistence-release-inventory:end --> |
| このスナップショット | [dsh-v0.1.5-alpha.2.schema.json](../../../../docs/persistence-changes/releases/dsh-v0.1.5-alpha.2.schema.json) |

このタグにおけるライター・バージョン定数のソース・エビデンス:

- `packages/core/session/src/types.ts`: `export const SESSION_FORMAT_VERSION = 3`

<a id="declaration"></a>
## 宣言

```yaml persistence-release
schemaVersion: 1
tag: dsh-v0.1.5-alpha.2
previous: dsh-v0.1.5-alpha.1
sessionFormatVersion: 3
changes:
  - root: event:deliverables/presented
    before: null
    after: 13d3d180f977bf78081d487ffa0ecb75857349bcab29a5a3fb48189fca2a6176
  - root: event:feedback/message-put
    before: 3b04fde0dc763cf84fbde7b6611b3194dd56d95d0c0bf0204311640468d586e1
    after: b5086d249e8502e9ead1d39156bb8d559bde7951cac0f14ce150345b4e42a2bf
  - root: event:feedback/record
    before: fb9df8180a202f3c845d5aa6a81697b2f7213f6735abc17535a55656be60a575
    after: b54940ff095c17e874c5be03815f4c2145a256cf3a1d34dae4ab2f7769dfffe8
  - root: event:subagent/catalog
    before: null
    after: ae1f7110feeec697b8cab42b68f7709aa7b3279764099dfb53c25890d2e5c871
```

<a id="changes"></a>
## 構造上の変更

<!-- persistence-release-changes:start -->

4 個の変更されたルートと 5 個の構造上の差分を検出した。以下のレビュー要件は比較のためだけに現在のルールで計算したものであり、歴史的な準拠、移行の正しさ、ランタイム互換性を主張するものではない。

| パス | 変更 | レビュー要件 |
|---|---|---|
| `event:deliverables/presented` | `root-added` | `not-required` |
| `event:feedback/message-put.data.item.category` | `optional-property-added` | `not-required` |
| `event:feedback/record.data.text` | `property-made-optional` | `not-required` |
| `event:feedback/record.data.category` | `optional-property-added` | `not-required` |
| `event:subagent/catalog` | `root-added` | `not-required` |

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
