---
kind: upgrade-guide
description: "Session projection の登録は保持された reader へ非同期に解決される。呼び出し可能な結果と registry レベルの状態読み取りは deprecated である。"
---
# Session projection 登録ハンドル

日本語 | [English](../../../../../docs/upgrade-guide/v0.2.1-rc.2/session-projection-handles/guide.md) | [中文](../../../../../docs/upgrade-guide/v0.2.1-rc.2/session-projection-handles/guide.zh.md)

> これは非公式の日本語訳です。英語版（[docs/upgrade-guide/v0.2.1-rc.2/session-projection-handles/guide.md](../../../../../docs/upgrade-guide/v0.2.1-rc.2/session-projection-handles/guide.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

## 変更

`@deepseek-ai/dsh-session-projection` は `SessionProjectionRegistry.register(definition)` を同期 disposer から `Promise<ProjectionHandle<S>>` に変更する。definition は同期的にインストールされ、結果を await すると `stateOf(session)` と `dispose()` を持つ handle が得られ、登録エラーはそれを reject する。runtime の値は呼び出し可能な thenable であり、ネイティブ Promise ではない。呼び出すと引き続き登録が解放されるが、その呼び出しは deprecated である。`typeof registration` は `"function"`、`registration instanceof Promise` は false であるため、これらのチェックで分岐するコードや関数を callback として扱うコードが影響を受ける。Vitest の `.rejects` は rejection を観測する代わりにそれを呼び出す。

registry レベルの `stateOf(session, key): unknown` は deprecated である。現在そのキーを登録している plugin がない場合、つまり owner が構成されていない、登録に到達していない、最後の登録を解放した場合は `undefined` を返す。`SessionProjectionStateMap` は削除され、host state 型は `ProjectionDefinition<K, S>` にローカルに残り、`stateSchema` から推論される。最後の登録が dispose された後、unit は snapshot、checkpoint、ライブ通知から外れるが、保持された handle はオンデマンドで Session event を fold し続ける。[Runtime compatibility](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-projection/README.md#deprecated-plugin-compatibility) が deprecated 呼び出しを詳述している。

`@deepseek-ai/dsh-agent` と `@deepseek-ai/dsh-subagent` は `sessionProjections` を必要とする。AgentRegistry は turn と step の状態を所有し、`ctx.agents.turnBoundaryOf(session)` を提供する。`@deepseek-ai/dsh-agent-loop` は `turnBoundaryProjectionDefinition` をエクスポートしなくなった。Claude Code mods bridge を含む turn-state consumer は `agents` を必要とする。preset と context-pressure の読み取りは `ctx.agentPresets.presetForSession(session)` と `ctx.tokenMeter.contextPressure(session)` を使う。API controller と mods bridge はそれらの定義を独立して登録しなくなった。所有サービスがない場合には、それらの optional-service 動作が適用される。`@deepseek-ai/dsh-api-session-controller` は `installModelSelectionProjection` を削除する。

## 移行

1. カスタム構成で `SessionProjectionRegistry` を `AgentRegistry`、`SubagentRuntime`、その他の projection owner より前にマウントする。plugin が `agents` を必要とする箇所には `@deepseek-ai/dsh-agent` を含める。
2. plugin 自身の定義については、非同期 `apply()` または `[Service.init]()` で `ctx.sessionProjections.register(definition)` を await し、依存する操作を公開する前に handle を保持する。`SessionProjectionStateMap` の拡張を削除し、state 型を `ProjectionDefinition<K, S>` の2番目の引数として渡す。client-value 宣言は `SessionProjectionMap` に残す。
3. `registry.stateOf(session, key)` を、`ctx.agents.turnBoundaryOf(session)` のような所有サービスのメソッド、または owner 内の `handle.stateOf(session)` に置き換える。それまでは `undefined` の結果を処理する。登録結果への呼び出しを await した `handle.dispose()` に置き換える。fiber の disposal は引き続き登録を自動的に解放する。
4. ネイティブ Promise が必要な場合は `Promise.resolve(registration)` を渡す。Vitest では `await expect(Promise.resolve(registration)).rejects.toThrow(...)` を含む。結果に対する `typeof` や `instanceof Promise` のチェックを await に置き換える。
5. `installModelSelectionProjection(ctx)` を `await ctx.sessionProjections.register(modelSelectionProjection)` に置き換え、定義を `@deepseek-ai/dsh-api-session-controller/src/model-selection-projection.ts` から import する。
6. カスタム plugin を typecheck し、activation、Session 状態の読み取り、拒否された登録、owner の disposal を確認する。
