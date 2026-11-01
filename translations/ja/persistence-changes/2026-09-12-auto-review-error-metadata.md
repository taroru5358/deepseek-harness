---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-09-12-auto-review-error-metadata

日本語 | [English](../../../docs/persistence-changes/2026-09-12-auto-review-error-metadata.md) | [中文](../../../docs/persistence-changes/2026-09-12-auto-review-error-metadata.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-09-12-auto-review-error-metadata.md](../../../docs/persistence-changes/2026-09-12-auto-review-error-metadata.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

永続化された PTC ディスパッチにオプショナルの構造化エラー・メタデータを、永続化されたネイティブ・ツール・エラーにオプショナルのユーザー向け理由を追加する。どちらの追加も Session 形式のバージョンを据え置く。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-09-12-auto-review-error-metadata
baseline: false
changes:
  - root: "event:tool/ptc-dispatch"
    previous: "2026-09-11-initial"
    after: "b5d66eaebed4da391b13498623b11142222149fbf5e025975dbc6d94f0d06796"
    decision: same-version
  - root: "event:tool/result"
    previous: "2026-09-11-initial"
    after: "3a803805bdeb805f32b229e399fb7258be8e32a89f89063a7c99957cfea942f7"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

既存の tool/ptc-dispatch イベントは error を省略でき、既存の tool/result のエラーは reason を省略できる。PTC ディスパッチ・イベントはモデル履歴に入らない。ネイティブのツール結果リプレイは data.message を投影し、エラー・メタデータを含めない。古いリーダーはこれらの追加を無視でき、モデルのリプレイは変わらない。現在の Web リーダーは理由の欠落を受け入れ、記録されたエラー同一性が一致する場合にのみ Auto review の拒否の詳細を表示する。Permission モードは既存のイベント内で文字列のままである。auto を選んでも、宣言された永続化型の変更は生じない。ヘッダー、イベント・エンベロープ、既存の値の型はいずれも変わらず、隣接する移行も不要である。

<a id="verification"></a>
## 検証

pnpm exec vitest run scripts/persistence-changes.spec.ts scripts/persistence-schema.spec.ts は 64 件のテストに合格した。対象を絞った permission、Auto review、ツール実行、agent-loop、subagent 継承、TypeScript SDK オーナーのテストは、ネイティブと PTC の拒否メタデータを含め、10 ファイルにわたる 390 件のテストに合格した。uv run --python 3.10 --group test --project python/sdk pytest python/sdk/tests/test_client.py -k preserves_auto_review_errors は 1 件のテストに合格した。永続化プレビューは 2 つのオプショナルな追加だけを分類し、バージョン引き上げを要求しなかった。

<a id="dev-note"></a>
## 開発メモ

なし。
