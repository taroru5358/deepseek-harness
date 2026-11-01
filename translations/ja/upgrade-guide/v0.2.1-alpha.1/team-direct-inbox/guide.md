---
kind: upgrade-guide
description: "Agent Teams は pending-message 上限を削除し、履歴上の queued mail の配信を停止する。"
---

# Agent Teams のメッセージは target inbox を使う

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/team-direct-inbox/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/team-direct-inbox/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/team-direct-inbox/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/team-direct-inbox/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

実験的な Agent Teams plugin は、別個の Team mailbox を保持する代わりに、新しいメッセージを target Agent の inbox へ直接送る。`send_message` は `{ "sent": true }` のみを返し、message ID も `accepted` や `queued` ステータスも公開しない。acceptance はモデル処理や同期 storage flush を確認するものではない。失敗した送信は再試行のために保持されず、明示的な再送は重複排除されない。

`maxPendingMessagesPerMember` 設定キーは削除された。履歴上の `team/message/queued` と `team/message/delivered` レコードは引き続き読めるが、アップグレード前に配信されなかった queued メッセージは決して dispatch されない。roster と task の状態は永続的なままである。

## 移行

1. アップグレード前に、pending の Team メッセージを完了させ、その target が受信したことを確認する。アップグレード後、必要な未配信メッセージを明示的に再送する。重複を避けるため、まず target の履歴を確認する。履歴 Session レコードを編集しない。
2. `cordis.yml` または overlay の `@deepseek-ai/dsh-experimental-agent-team` plugin 設定から `maxPendingMessagesPerMember` を削除する。Team queue の代替上限はない。`maxMessageBytes` は引き続き、送信者フレーム付きの完全なメッセージごとに上限する。
3. `send_message` 結果の consumer を、message ID や delivery status の代わりに `sent` を読むように更新し、送信エラーを明示的に処理する。成功を target の作業完了として扱わない。`ctx.agentTeams.sendMessage()` サービスは引き続き `messageId` を返す。
4. Team 対応 profile を起動し、別のメンバーにメッセージを送り、tool が `{ "sent": true }` を返し、target がそれを受信することを確認する。delivery のセマンティクスについては [パッケージリファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/agent-team/README.md#use-this-package) を参照。
