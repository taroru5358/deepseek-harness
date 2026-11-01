---
kind: upgrade-guide
description: "session-title provider は直接登録し、自身の戦略設定を所有し、準備済みの実行のみを共有する。"
---

# session-title provider を移行する

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/session-title-provider-strategy/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/session-title-provider-strategy/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-alpha.1/session-title-provider-strategy/guide.md](../../../../../docs/upgrade-guide/v0.2.1-alpha.1/session-title-provider-strategy/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-session-title-llm` は `registerSessionTitleLlmProvider`、`generateSessionTitleWithLlm`、`SessionTitleLlmMessageSelector` をエクスポートしなくなった。`SessionTitleLlmConfig` は実行制御（`maxInputBytes`、`maxOutputTokens`、`timeoutMs`、任意の `provider`/`model` ペア）のみを保持するようになり、`targetWords` と `targetCjkCharacters` は各 provider 自身の `Config` に移った。各 provider は自身の system instruction、message framing、title parsing、reasoning selection、output interpretation を所有し、`ctx.sessionTitle.register()` で直接登録し、`executeSessionTitleLlm(ctx, config, request, providerId, { system, input, messageSeqs, selectReasoningEffort })` を呼ぶ。この関数は組み立てられた content block、終端 finish、使用された model route を返す。`SessionTitleProviderRequest` は任意の `currentTitle` snapshot を得る。

## 移行

1. `registerSessionTitleLlmProvider(...)` 呼び出しを直接の `ctx.sessionTitle.register({ id, automatic, generate })` に置き換える。同じ `id` と cadence を保つ。
2. `generate(request)` で独自の system prompt と user input を構築し、`selectReasoningEffort` 関数を渡してから、`{ system, input, messageSeqs, selectReasoningEffort }` で `executeSessionTitleLlm` を呼ぶ。
3. `tool-calls` または `max-tokens` の終端 finish と tool-call block を拒否し、text block から title を導出する。
4. 戦略が承認済みの provider title を保持すべき場合は `request.currentTitle` を読む。title がない前は存在しない。
5. title を生成し、保存された結果と正確な `session/title-llm-request` レコードを確認する。
