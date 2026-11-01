---
kind: upgrade-guide
description: "JobId は registry のライフタイムをまたいで1つのバックグラウンド実行を識別する。"
---

# バックグラウンド job id は1つの実行を識別する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/job-id-identity/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/job-id-identity/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/job-id-identity/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/job-id-identity/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

job registry は `<kind>-N` の代わりに `<kind>-<short-uuid>` を返す。サフィックスは16文字の小文字 UUID 16進文字である。同じ `JobId` がライブ制御、`job/registered`、`job/settled` に現れる。ライフサイクルレコードは `executionId` を含まなくなった。識別子は引き続き所有者の認可を必要とする。既存の Session 履歴は変わらず、その job を resume しない。

## 移行

1. start 結果または `jobs.list()` が返す正確な id を `job_output` と `job_kill` に渡す。待機には `wait: true` 付きの `job_output` を使う。kind と序数から id を構築するコードを削除する。
2. SDK lifecycle event を `data.jobId` で pair する。`JobExecutionId` の import と、それをプロセスローカルの `jobId` にマップするコードを削除する。
3. 新しい job の登録と settlement が同じ `jobId` を含み、後の registry インスタンスが異なるものを発行することを確認する。
