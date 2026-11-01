---
kind: upgrade-guide
description: "AccountBonusNotification は検証されない任意メタデータを公開し、expiry のない通知を許可する。"
---

# account 通知の consumer を更新する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.4/account-notification-fields/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.4/account-notification-fields/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.4/account-notification-fields/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.4/account-notification-fields/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`DeepSeekAccount.getUnnotifiedBonuses()` と `account/getUnnotifiedBonuses` が返す公開の `AccountBonusNotification` は、`campaign`、`amount`、`currency`、`grantedAt` を、`JsonValue` 型の任意で検証されないメタデータとして保持する。通知は `orderId`、`message`、`expiresAt` を維持し、任意の `title` を追加する。空の `expiresAt` は通知が期限切れにならないことを意味する。`orderId` は UUID を要求せず、任意の文字列を受け付ける。

## 移行

1. 任意のメタデータ値を使う前に絞り込む。その存在と形式はもはや保証されない。`message` を表示し、`title` がない場合は client のデフォルトタイトルを使う。
2. タイムスタンプを parse する前に、空の `expiresAt` を期限なしとして扱う。通知を acknowledge する際は `orderId` を変更しない。
3. provider fixture を現在のフィールドを使うよう更新する。UUID でない ID と空の expiry を持つ通知が表示され、acknowledge できることを確認する。
