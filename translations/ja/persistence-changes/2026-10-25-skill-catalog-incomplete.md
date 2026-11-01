---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-10-25-skill-catalog-incomplete

日本語 | [English](../../../docs/persistence-changes/2026-10-25-skill-catalog-incomplete.md) | [中文](../../../docs/persistence-changes/2026-10-25-skill-catalog-incomplete.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-10-25-skill-catalog-incomplete.md](../../../docs/persistence-changes/2026-10-25-skill-catalog-incomplete.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

検出が現在読めるスキルだけを返す場合に、skill-catalog メッセージ・ソースへオプショナルの incomplete: true メタデータを追加する。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-10-25-skill-catalog-incomplete
baseline: false
changes:
  - root: "event:agent/inbox/spliced"
    previous: "2026-10-22-action-card-status"
    after: "9e8043098fb89db6d7c92a92c70146c5c8d24fa88f733c687fa29e5e7a58ab30"
    decision: same-version
  - root: "event:developer/message"
    previous: "2026-10-22-action-card-status"
    after: "95acdbd066cf445d37092e5fdfe2e8717ef6ca1aed50f70e2e6037a6989a1f2f"
    decision: same-version
  - root: "event:session/title-llm-request"
    previous: "2026-10-22-action-card-status"
    after: "eaa9125aec0fb28478096c46e73e21bf512ab0d84d826bfb41e7308707734cb5"
    decision: same-version
  - root: "event:user/message"
    previous: "2026-10-22-action-card-status"
    after: "39adb7fce8475af6402acd925ea79697b3341c22422ac32833d71217d84e987f"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

既存のカタログはこのフラグを省略し、完全なままである。新しいリーダーはエントリーと完全性の両方を比較する。古いリーダーは変更されていないソース判別子とエントリー・フィールドを受け入れ、メッセージ内容に既に描画されている検出警告を保持し、オプショナルのフラグを無視する。イベント判別子、必須フィールド、Session 形式のバージョンはいずれも変わらない。

<a id="verification"></a>
## 検証

対象を絞った skill-filesystem と tool-skill のテスト: 71 件に合格し、変更された両方のソース・ファイルを 100% カバーした。キー不要のヘッドレスな skill-root-isolation と skill-load のスナップショット: 2 件に合格した。分離のフィクスチャは部分的なカタログを記録し、失敗したルートがあっても読めるスキルをロードする。

<a id="dev-note"></a>
## 開発メモ

なし。
