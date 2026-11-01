---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-10-05-ptc-operation-metadata

日本語 | [English](../../../docs/persistence-changes/2026-10-05-ptc-operation-metadata.md) | [中文](../../../docs/persistence-changes/2026-10-05-ptc-operation-metadata.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-10-05-ptc-operation-metadata.md](../../../docs/persistence-changes/2026-10-05-ptc-operation-metadata.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

履歴が操作時点のディレクトリとファイル・ターゲットを保持できるよう、永続化されたネストした PTC ツール結果にオプショナルの純粋な表示用メタデータを追加する。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-10-05-ptc-operation-metadata
baseline: false
changes:
  - root: "event:tool/ptc-dispatch"
    previous: "2026-09-16-session-format-v4"
    after: "d66e11dddf5dfa3c71547e2cbd7b0608824cd8d6243fc16f532017ff28e8fba5"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

meta を持たない既存の tool/ptc-dispatch レコードは有効なままである。オプショナルの JsonValue フィールドは、正規のプログラム値、モデルに見える内容、ツールの同一性、メッセージの順序を変えない。記録された相対ターゲットを持たない履歴上の呼び出しは、安全な利用不可のナビゲーションを保つ。ライターと承認済みベースラインは V4 のままである。

<a id="verification"></a>
## 検証

対象を絞った 963 件のテストに合格し、影響を受ける 6 つのランタイム/UI ソースをファイルごとに 100% カバーした。実際のネストしたシェルと相対ファイルシステムのリグレッション、凍結されたメタデータの対照、TypeScript SDK の 45 件、Python SDK の 29 件、および所有する 3 つのキー不要のリフレッシュ/リプレイが合格した。

<a id="dev-note"></a>
## 開発メモ

なし。
