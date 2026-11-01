---
description: "永続化型の遷移とその互換性の承認を記録する。"
kind: persistence-change
---

# 2026-09-30-job-session-lifecycle

日本語 | [English](../../../docs/persistence-changes/2026-09-30-job-session-lifecycle.md) | [中文](../../../docs/persistence-changes/2026-09-30-job-session-lifecycle.zh.md)

> これは非公式の日本語訳です。英語版（[docs/persistence-changes/2026-09-30-job-session-lifecycle.md](../../../docs/persistence-changes/2026-09-30-job-session-lifecycle.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 概要

一意の JobId、元のタイムスタンプ、作成したツール呼び出し、プロデューサーの完了報告を伴う、所有ジョブの登録と確定を記録する。

## 目次

- [宣言](#declaration)
- [互換性](#compatibility)
- [検証](#verification)
- [開発メモ](#dev-note)

<a id="declaration"></a>
## 宣言

```yaml persistence-change
schemaVersion: 1
id: 2026-09-30-job-session-lifecycle
baseline: false
changes:
  - root: "event:job/registered"
    previous: null
    after: "f5520c1d261ca86ea03735799c311a0a9ede8babb588cf0a65422576d2ee0ad1"
    decision: same-version
  - root: "event:job/settled"
    previous: null
    after: "edd37b1e4974af648633014435572089b1e8fb71910468a0767a7e28fdd22abf"
    decision: same-version
```

<a id="compatibility"></a>
## 互換性

Session エンベロープを変えずに、必須のジョブ・イベント種別を追加する。新しい JobId は、ライブ制御と永続的なペアリングの両方に種別接頭辞付きの UUID を使う。履歴上のセッションに、捏造されたジョブ履歴が与えられることはない。古いビルドは未知の必須イベントを拒否する。producerReported は完了約束の履行を表し、検証済みのリソース解放を表すものではない。

<a id="verification"></a>
## 検証

レジストリとスナップショット正規化器のテストが、因果順序と一意の JobId をカバーする。CLI と SDK のスナップショットは、実行中の登録、確定、通知を検証する。[ジョブ・ライフサイクルの契約](../subsystems/jobs.md#session-lifecycle-records)は、アプリケーションの終了処理中に不完全な終端履歴を許容する。

<a id="dev-note"></a>
## 開発メモ

なし。
