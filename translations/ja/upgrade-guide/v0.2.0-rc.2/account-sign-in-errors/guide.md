---
kind: upgrade-guide
description: "account サインインの transport 失敗は network ではなく no-response を報告するようになった。"
---

# account サインインのエラーコード

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.0-rc.2/account-sign-in-errors/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.0-rc.2/account-sign-in-errors/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.0-rc.2/account-sign-in-errors/guide.md](../../../../../docs/upgrade-guide/v0.2.0-rc.2/account-sign-in-errors/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`SignInErrorCode` は `no-response` を追加する。`account/getState`、`account/startSignIn`、`account/watch` が返す account 状態は、Response を返す前に fetch が失敗した場合（リクエスト timeout を含む）にこのコードを報告する。これらの失敗は以前 `network` を報告していた。HTTP response エラーは引き続き `network` を報告し、authorization attempt の期限切れは引き続き `expired` を報告する。

## 移行

1. account API または `@deepseek-ai/dsh-deepseek-account` の client で、`SignInAttemptView.errorCode` の validator と網羅的 handler に `no-response` を追加する。
2. `no-response` にはネットワークトラブルシューティングを表示する。`network` の汎用失敗処理は維持する。これは HTTP と account 詳細取得の失敗もカバーする。
3. `errorCode: 'no-response'` のログイン失敗が client に受け入れられ、再試行を提供することを確認する。HTTP エラー応答は汎用失敗処理を維持する。
