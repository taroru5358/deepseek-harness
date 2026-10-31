# エージェントのターンとステップのライフサイクル

日本語 | [English](../../docs/agent-lifecycle.md) | [中文](../../docs/agent-lifecycle.zh.md)

> これは非公式の日本語訳です。英語版（[docs/agent-lifecycle.md](../../docs/agent-lifecycle.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。
>
> Mermaid 図のイベント名・参加者名はコード識別子との対応を保つため原文のままです。図の生成元は `scripts/gen-doc-graphs.ts` です。

このシーケンスは、[architecture.md](architecture.md#ターン・フロー)の視覚的な補助資料です。永続的なリプレイ事実は `session/event` に、ライブな制御・状態は `agent/*` に置きます。

```mermaid
sequenceDiagram
  participant User
  participant Agent
  participant Driver
  participant Hooks as hook listeners
  participant Prompt as ctx.systemPrompt
  participant LLM as ctx.llm
  participant Tools as ctx.tools
  participant Session
  participant SDK as UI or SDK listener
  User->>Agent: followup(content)
  Agent-->>SDK: <code>agent/inbox/spliced</code>
  Agent-->>SDK: <code>agent/inbox/inserted</code> { message }
  Agent->>Driver: queued work wakes driver
  Driver-->>SDK: <code>agent/status</code> running
  Driver->>Session: <code>turn/start</code>
  Note over Agent,Driver: claim pending next-step input plus one queued prompt
  Driver-->>SDK: <code>agent/inbox/spliced</code> pure deletion
  Driver-->>SDK: <code>agent/inbox/claimed</code> { message, turn } per message
  Driver->>Tools: resolve({ agent, modelSelection, signal }) once per proposed step
  Driver->>Prompt: <code>system-prompt/assemble</code> waterfall with the step tool set
  Driver->>Hooks: <code>agent/pre-step</code> waterfall
  Hooks-->>Driver: authoritative reject or enter(messages)
  alt proposed step rejected, first batch empty, or pre-step failed
    Driver-->>Driver: claimed batch stays removed, the open turn spends no step
  else enter proposed step
  Driver->>Session: <code>step/start</code>
  Driver->>Hooks: <code>agent/request</code> waterfall
  Driver->>LLM: prepareCall(config, signal)
  Note over Driver,LLM: cancellation during either async phase commits neither system nor users
  Note over Driver,Session: synchronous admission using the prepared call capability
  Driver->>Session: <code>system/message</code> ordered per-node reconciliation
  Driver->>Session: <code>user/message</code> per entered message
  Driver->>Session: <code>request/header</code> and <code>request/context</code> as needed
  Driver->>Driver: derive and freeze request from the log
  Driver->>LLM: bound prepared call through <code>llm/stream</code> waterfall
  LLM-->>Driver: StreamChunk*
  Driver-->>SDK: <code>agent/assistant-stream</code> chunk*
  alt final adapter or terminal in-band request failure
    Driver->>Session: <code>assistant/attempt</code>
    Driver-->>SDK: <code>agent/assistant-stream</code> committed end
    Driver->>Hooks: <code>agent/request-error</code> waterfall
    Hooks-->>Driver: return retry action or preserve the original error
    Note over Driver,LLM: retry in the open step: prepare and reconcile the same rendered assembly without repeating pre-step or users
  else model request succeeded
  Driver->>Session: <code>assistant/message</code>
  Driver-->>SDK: <code>agent/assistant-stream</code> committed end
  Driver->>Tools: classify pending call by executionMode
  loop barriers and bounded rolling pool, reclassify before start
    opt call starts
      Driver->>Session: <code>tool/call</code>
      Driver->>Tools: ordered pre, concurrent execute
      Tools-->>Session: tool-owned events when applicable
    end
    opt next model-order result ready
      Driver->>Tools: ordered post
      Driver->>Session: <code>tool/result</code>
    end
  end
  Driver->>Session: <code>step/end</code>
  Driver->>Tools: dispose the step tool set
  opt natural stop and next-step inbox empty
    Driver->>Hooks: <code>agent/turn-stopping</code> serial terminal checkpoint
  end
  opt next-step input is pending
    Driver-->>Driver: claim pending next-step input
    Driver-->>SDK: <code>agent/inbox/claimed</code> { message, turn } per message
    Driver->>Hooks: <code>agent/pre-step</code> waterfall
    Hooks-->>Driver: authoritative reject or enter(messages)
  end
  end
  end
  Driver->>Session: <code>turn/end</code>
  Driver-->>SDK: <code>agent/status</code> idle
```

`assistant/message` イベントは、内容なしの完了や `max-tokens` での完了を含む、すべての成功したプロバイダー呼び出しを記録し、正確なコンパクトな timed stream を埋め込みます。空の内容は派生履歴に残りません。失敗、リトライ、キャンセル、ストリーム・エラーの試行が、surface メッセージなしで決着に到達した場合は、そのストリームを `assistant/attempt` として記録します。ライブな `agent/assistant-stream` の chunk フレームは一時的です。リプレイはどちらかの永続的な決着を読み、決着前のハードなプロセス損失は永続的な attempt ストリームを残しません。

`dsh-compaction-basic` は、リクエスト派生前の圧力に対して `agent/pre-step` を、正規のコンテキスト溢れに対してのみ `agent/request-error` を使います。どちらかのトリガーが条件を満たすと、オプションのツール結果 pruning が要約選択の前に実行されます。回復は開いているステップ内で実行され、pruning または要約が surface 置換の世代を進めた場合にのみリトライします。それ以外の場合は、元のリクエスト・エラーが権威あるものとして残ります。各リトライは、組み立て、pre-step、ユーザー受付を繰り返さずに、呼び出しを準備し、保持されたレンダリング済み組み立てをリクエスト派生前に調整します。

返された `agent/pre-step` の判断は権威があります。`next()` をラップするリスナーは、置換が意図されていない限り、下流のメッセージと `startsRequestSeries` を保持します。ステアリングと注入されたコンテキストは、後の claim 操作が次のステップ・バッチを取得した後、同じウォーターフォールを通ります。

リプレイ可能な transcript データを必要とする SDK ユーザーは `session/event` を消費してください。`agent/*` は、キュー・状態、プロンプト傍受、リクエスト構築、ステアリング、継続、エラーのためのライブ・コーディネーション API です。

メンテナンス・モード: 厳選された Mermaid シーケンスです。正確なイベント署名は、生成された Cordis カタログにあります。
