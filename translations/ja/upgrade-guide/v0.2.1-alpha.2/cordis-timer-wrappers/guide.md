---
kind: upgrade-guide
description: "timer wrapper は TypeScript で void を返し、disposal 後の呼び出しを無視する。"
---

# timer wrapper の戻り値型と disposal

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/cordis-timer-wrappers/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/cordis-timer-wrappers/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.2/cordis-timer-wrappers/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.2/cordis-timer-wrappers/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`ctx.throttle()` と `ctx.debounce()` が返す関数は `void` 戻り値型を宣言するようになった。以前は callback の戻り値型を継承していたが、runtime ではすでに `undefined` を返していた。wrapper の戻り値を使ったり、callback の型に代入したりする TypeScript コードは更新が必要な場合がある。Host と browser は同じ `TimerService` 実装を使う。

`.dispose()` を呼ぶか、所有する fiber を dispose すると、pending の作業がキャンセルされ、以降の wrapper 呼び出しは不活性になる。以前は dispose された throttle wrapper が leading 呼び出しを実行できた。debounce wrapper はすでに disposal 後の呼び出しを無視していた。

`@deepseek-ai/dsh-cordis-client-runner/client` エントリは `ClientTimerService` をエクスポートしなくなった。runner は `@deepseek-ai/cordis-plugin-timer` の共有 `TimerService` をインストールする。

## 移行

1. TypeScript に wrapper 型を推論させるか、必要に応じて `.dispose()` を保持しつつ、呼び出しシグネチャを `void` 戻り値型で宣言する。
2. callback の結果は callback 内で消費する。wrapper を await しても、非同期 callback の完了は待たない。
3. 必要な作業を disposal の前に移すか、active な plugin で新しい wrapper を作成する。
4. runner からの `ClientTimerService` の import を削除する。consumer plugin は `inject: ['timer']` を宣言して `ctx.timer` を使う。サービスをインストールするコードは `@deepseek-ai/cordis-plugin-timer` から `TimerService` を import する。
5. extension の型チェックを実行し、disposal 後の呼び出しが callback を呼び出さず、作業もスケジュールしないことを確認する。
