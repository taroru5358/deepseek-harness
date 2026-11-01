---
kind: upgrade-guide
description: "Session follow の呼び出し元は、バックグラウンドの Agent activation を明示的に要求しなければならない。"
---
# Session を follow するときに activation を要求する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.3/session-follow-activation/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.3/session-follow-activation/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.3/session-follow-activation/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.3/session-follow-activation/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`session.follow` は `activate` が省略された場合、コールドな通常 Agent を自動的に resume しなくなった。opening snapshot を読み、activation なしで後続の event を follow する。履歴ストリームを開くことで Agent を利用可能にしていた統合が影響を受ける。出荷される Web Client は `activate: true` を明示的に送り、従来の opening 動作を維持する。

`activate: true` の場合、activation は opening snapshot が計算された後に、それが配信される前に開始される。その snapshot は Agent の準備完了を acknowledge しない。subagent Session は引き続き通常の Agent activation を使わない。

## 移行

1. コールドな通常 Agent を resume する必要がある `session.follow` リクエストに `activate: true` を追加する。既存の `address` と履歴ウィンドウのフィールドは保持する。
2. 受動的な viewer では `activate` を省略するか `false` に設定する。
3. activation 有効のストリームが引き続き opening snapshot を返すことを確認し、その後 Agent の可用性を別途確認する。snapshot の受信を activation の完了として扱わない。

履歴と状態の監視動作については [Session Controller リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/session-controller/README.md) を参照。
