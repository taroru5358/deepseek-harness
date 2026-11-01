---
description: "dsh-v0.1.2-alpha.4 における回顧的な Session 永続化型と隣接リリースの変更。"
kind: persistence-release
---

# 永続化リリース: dsh-v0.1.2-alpha.4

日本語 | [English](../../../../docs/persistence-changes/releases/dsh-v0.1.2-alpha.4.md) | [中文](../../../../docs/persistence-changes/releases/dsh-v0.1.2-alpha.4.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/releases/dsh-v0.1.2-alpha.4.md](../../../../docs/persistence-changes/releases/dsh-v0.1.2-alpha.4.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

論理的な SessionHeader はオプショナルの seedLength を必須の isSeeded に置き換えるが、物理的な JSONL ヘッダーは引き続き seedLength を宣言する。subagent-report と coordinator のユーザー・メッセージ・ソース・バリアントは agent-message になる。ライター形式は 0 のままである。

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
| ソース・タグ | `dsh-v0.1.2-alpha.4` |
| ソース日付 | 2026-09-01T15:37:26.000Z |
| リリース記録 | リリース・オブジェクトあり。 |
| 前回のリリース | [dsh-v0.1.2-alpha.3](dsh-v0.1.2-alpha.3.md) |
| Session ライター・バージョン | 0 |
| 再構成されたインベントリー | <!-- persistence-release-inventory:start -->54 ルート / 415 型<!-- persistence-release-inventory:end --> |
| このスナップショット | [dsh-v0.1.2-alpha.4.schema.json](../../../../docs/persistence-changes/releases/dsh-v0.1.2-alpha.4.schema.json) |

このタグにおけるライター・バージョン定数のソース・エビデンス:

- `packages/core/session/src/types.ts`: `export const SESSION_FORMAT_VERSION = 0`

<a id="declaration"></a>
## 宣言

```yaml persistence-release
schemaVersion: 1
tag: dsh-v0.1.2-alpha.4
previous: dsh-v0.1.2-alpha.3
sessionFormatVersion: 0
changes:
  - root: SessionHeader
    before: a50373168c4935222b1681223d919a56d095ce37ad45c5ba2c20d75235adf937
    after: 001ed6f66d67fac9cb9594b55f13557db94174fe5d5954789bb5a2d6d5927226
  - root: event:agent/inbox/spliced
    before: ee796690277eafbcda4437478de7a8f01d983424d6238fe472df9b3b0d97a89f
    after: 15cdff6391d58ea00d7e2fe113b663d5479cbb3fd1af26717ad933c770bca081
  - root: event:session/title-llm-request
    before: 61651f4ca07ab9ebef773d82fafbcb74d39190a255c69ecc36084046123a9cb4
    after: 2cfb71f7819bc88a6bccbffa8b7ae5664233af6f6e1dfb068db25e2e17ddd8c7
  - root: event:user/message
    before: e23368db1646a9ac10d2bd4629084fdff583a1db2c83ffaa3f2f201d0c45f3e4
    after: e950c87ba49bd8175b8a670b319a599d5ed14cde996540d5ba91a141fad68781
```

<a id="changes"></a>
## 構造上の変更

<!-- persistence-release-changes:start -->

4 個の変更されたルートと 5 個の構造上の差分を検出した。以下のレビュー要件は比較のためだけに現在のルールで計算したものであり、歴史的な準拠、移行の正しさ、ランタイム互換性を主張するものではない。

| パス | 変更 | レビュー要件 |
|---|---|---|
| `SessionHeader.seedLength` | `property-removed` | `review-required` |
| `SessionHeader.isSeeded` | `required-property-added` | `review-required` |
| `event:agent/inbox/spliced.data.inserted[].source` | `union-variants-changed` | `review-required` |
| `event:session/title-llm-request.data.messages[].source` | `union-variants-changed` | `review-required` |
| `event:user/message.data.source` | `union-variants-changed` | `review-required` |

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
