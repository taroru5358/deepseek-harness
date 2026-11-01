---
kind: upgrade-guide
description: "DeepSeekAccount.watch は明示的に要求されない限り Platform identity の変更を含まなくなった。"
---

# account identity の変更を明示的に購読する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/deepseek-account-identity-watch/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/deepseek-account-identity-watch/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.2/deepseek-account-identity-watch/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/deepseek-account-identity-watch/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

公開されている `DeepSeekAccount.watch(signal)` メソッドは以前、profile 読み取りが初めてキャッシュされた Platform user ID を提供または変更したときに snapshot を emit していた。現在はデフォルトで account-state の変更のみを emit する。このストリームを使って `getPlatformSession()` を refresh する plugin は、identity 変更をオプトインしなければならない。credential の置き換えとサインイン状態の変更は引き続きすべての subscriber に通知する。

## 移行

1. Platform session を refresh したり、その user ID を同期したりする plugin で `deepseekAccount.watch(signal)` の呼び出しを見つける。
2. それらの呼び出しを `deepseekAccount.watch(signal, true)` に変更する。サインインと credential 状態のみを必要とする account UI の subscription は変更しない。
3. ready な `getProfile()` の結果で user ID が初めて現れるか変更されると、subscription が起動して `getPlatformSession()` を再読取することを確認する。同じ ID での繰り返し読み取りは、別の identity 通知を emit しない。
