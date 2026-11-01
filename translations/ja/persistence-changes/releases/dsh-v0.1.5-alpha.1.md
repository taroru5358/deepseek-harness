---
description: "dsh-v0.1.5-alpha.1 における回顧的な Session 永続化型と隣接リリースの変更。"
kind: persistence-release
---

# 永続化リリース: dsh-v0.1.5-alpha.1

日本語 | [English](../../../../docs/persistence-changes/releases/dsh-v0.1.5-alpha.1.md) | [中文](../../../../docs/persistence-changes/releases/dsh-v0.1.5-alpha.1.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/releases/dsh-v0.1.5-alpha.1.md](../../../../docs/persistence-changes/releases/dsh-v0.1.5-alpha.1.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

ライター形式は 2 から 3 へ進む。system/message が追加され、EpochHeader は system を削除する。サーフェス置換のフィールドは start/end から startSeq/endSeq へ変わり、tool/code-dispatch のイベント・キーは tool/ptc-dispatch のキーになる。

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
| ソース・タグ | `dsh-v0.1.5-alpha.1` |
| ソース日付 | 2026-09-08T15:25:45.000Z |
| リリース記録 | リリース・オブジェクトあり。 |
| 前回のリリース | [dsh-v0.1.3-alpha.2](dsh-v0.1.3-alpha.2.md) |
| Session ライター・バージョン | 3 |
| 再構成されたインベントリー | <!-- persistence-release-inventory:start -->57 ルート / 443 型<!-- persistence-release-inventory:end --> |
| このスナップショット | [dsh-v0.1.5-alpha.1.schema.json](../../../../docs/persistence-changes/releases/dsh-v0.1.5-alpha.1.schema.json) |

このタグにおけるライター・バージョン定数のソース・エビデンス:

- `packages/core/session/src/types.ts`: `export const SESSION_FORMAT_VERSION = 3`

<a id="declaration"></a>
## 宣言

```yaml persistence-release
schemaVersion: 1
tag: dsh-v0.1.5-alpha.1
previous: dsh-v0.1.3-alpha.2
sessionFormatVersion: 3
changes:
  - root: SessionEventEnvelope
    before: 04184edc061905410cc7e3db8d5c7cf393d739c89eb523ba8bad3831fb23ba95
    after: 5776e5553ff2dfe3f5bc202dbb1e7c9f93e35a531aebb7764c23b2b6153b2ccc
  - root: SessionHeader
    before: 15918d785bdccedf0dba204622c75591bfb833bbf107ae0894bbc44cfbd73c1d
    after: 2a8b114637e66faab5fae1c6e2fba65320db6224c80f5f36c5bf32f937154d03
  - root: event:assistant/message
    before: 83f26c7a856f772e6ffd9a5a144e5aaccf771ac54cef41948f99eed99e1a25a7
    after: a58ca1bed502d425a8b65cac02f59717498cd296de9513c6f81b83b791c20e9b
  - root: event:request/context
    before: f6b733e38d46dde3f8ab1ade2be3362abb6642eb97982a98ee64a57ff60d0c28
    after: 37cbc9cf06d494cfe5c67f078af1605eacb8e4c2c853e9e26455b73de0ee7ccf
  - root: event:request/header
    before: 1de6c9c8764045ea9a1357bae6ccbbd57c19f6f337f8f25697c919c7cc9abf16
    after: 8ea13bcd983eab596dbfd41c5c2b62a0b318b71cd2a987a6fd752c6ced7e4489
  - root: event:system/message
    before: null
    after: 0ae7ccdc1cc5d9d43772ebd5c7357ff509011c3a6ba188d9c274d33df2542c42
  - root: event:tool/code-dispatch
    before: f8dc5624cd0c942a069063ae0d90a61d821b4fddd1d967dcded91cf17933e708
    after: null
  - root: event:tool/code-dispatch-start
    before: 9eb21c10fc675e1fa4184e5eecc9697aa87054ffd836d49428174897f0e64b62
    after: null
  - root: event:tool/ptc-dispatch
    before: null
    after: da807583a19f4ebe2bc9115c9686016e1521e405bdfaced296ae652b91e8e5f6
  - root: event:tool/ptc-dispatch-start
    before: null
    after: ec38b5949af8eacaf00df002f4acbe344f934f8a061e9cdc65a52a48e5f6dd93
  - root: event:tool/result
    before: eacefd48000720582725ccfc6ff7a0ac02755a1a3c57a8f9754125f99aab6c1b
    after: cc3e270ab982cbafff602b753647b6f056decb7fe4fd523afd2f157a65c3d622
  - root: event:user/message
    before: a57a77f18de7b37b724293830a57064696895d69f2afcd2c0264f41cd13ba980
    after: 2e49ede7fb3a592cce8919ebe982f6ca1c1f67abe4def359ffc028c85e6d1ad8
```

<a id="changes"></a>
## 構造上の変更

<!-- persistence-release-changes:start -->

12 個の変更されたルートと 25 個の構造上の差分を検出した。以下のレビュー要件は比較のためだけに現在のルールで計算したものであり、歴史的な準拠、移行の正しさ、ランタイム互換性を主張するものではない。

| パス | 変更 | レビュー要件 |
|---|---|---|
| `SessionEventEnvelope` | `union-variants-changed` | `review-required` |
| `SessionHeader.version` | `type-changed` | `review-required` |
| `event:assistant/message.sourceEventSeqs` | `property-removed` | `review-required` |
| `event:assistant/message.surfaceOp` | `property-made-required` | `review-required` |
| `event:assistant/message.surfaceOp.end` | `property-removed` | `review-required` |
| `event:assistant/message.surfaceOp.start` | `property-removed` | `review-required` |
| `event:assistant/message.surfaceOp.endSeq` | `required-property-added` | `review-required` |
| `event:assistant/message.surfaceOp.startSeq` | `required-property-added` | `review-required` |
| `event:request/context.data.systemPromptUpdate` | `optional-property-added` | `not-required` |
| `event:request/header.data.header.system` | `property-removed` | `review-required` |
| `event:system/message` | `root-added` | `review-required` |
| `event:tool/code-dispatch` | `root-removed` | `review-required` |
| `event:tool/code-dispatch-start` | `root-removed` | `review-required` |
| `event:tool/ptc-dispatch` | `root-added` | `not-required` |
| `event:tool/ptc-dispatch-start` | `root-added` | `not-required` |
| `event:tool/result.surfaceOp` | `property-made-required` | `review-required` |
| `event:tool/result.surfaceOp.end` | `property-removed` | `review-required` |
| `event:tool/result.surfaceOp.start` | `property-removed` | `review-required` |
| `event:tool/result.surfaceOp.endSeq` | `required-property-added` | `review-required` |
| `event:tool/result.surfaceOp.startSeq` | `required-property-added` | `review-required` |
| `event:user/message.surfaceOp` | `property-made-required` | `review-required` |
| `event:user/message.surfaceOp.end` | `property-removed` | `review-required` |
| `event:user/message.surfaceOp.start` | `property-removed` | `review-required` |
| `event:user/message.surfaceOp.endSeq` | `required-property-added` | `review-required` |
| `event:user/message.surfaceOp.startSeq` | `required-property-added` | `review-required` |

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
