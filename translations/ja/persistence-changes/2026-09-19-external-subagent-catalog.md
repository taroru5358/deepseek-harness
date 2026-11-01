---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-09-19-external-subagent-catalog

日本語 | [English](../../../docs/persistence-changes/2026-09-19-external-subagent-catalog.md) | [中文](../../../docs/persistence-changes/2026-09-19-external-subagent-catalog.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-09-19-external-subagent-catalog.md](../../../docs/persistence-changes/2026-09-19-external-subagent-catalog.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

ローカル Session を持たない子のために、親カタログのエントリーへ external モードを追加する。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-09-19-external-subagent-catalog
baseline: false
changes:
  - root: "event:subagent/catalog"
    previous: "2026-09-20-unknown-child-catalog"
    after: "53750b9abf61efb7992b0aadc3e247ec927449c63266df28230fd1139f5590ac"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

既存のバージョン 0 とバージョン 1 のワンショットおよび継続可能なファクトは有効なままである。バージョン 1 は unknown モードも保持する。新しい external 実行は、別個のマーカーなしで mode external を持つペイロード・バージョン 2 を書き込む。作成は 1 つのメンバーシップ・ファクトを書き込み、実行とクリーンアップはそれを更新しない。重複した子 id は引き続き無効である。これはリリース済みの世代や Session 形式のバージョンを変えない。

<a id="verification"></a>
## 検証

対象を絞ったカタログ、external の有効化、ネイティブ V4 の復元、クライアントのナビゲーション、制御ツールのテストが合格した。これらは external のメンバーシップ、履歴上のローカル・モード、不正なマーカーの拒否、ナビゲート不能な external リーフ、重複した子の拒否をカバーする。

<a id="dev-note"></a>
## 開発メモ

なし。
