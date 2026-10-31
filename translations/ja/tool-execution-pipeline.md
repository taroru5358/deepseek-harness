# ツール実行パイプライン

日本語 | [English](../../docs/tool-execution-pipeline.md) | [中文](../../docs/tool-execution-pipeline.zh.md)

> これは非公式の日本語訳です。英語版（[docs/tool-execution-pipeline.md](../../docs/tool-execution-pipeline.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。
>
> Mermaid 図のイベント名・識別子はコードとの対応を保つため原文のままです。図の生成元は `scripts/gen-doc-graphs.ts` です。

組み込みエグゼキューターは、生バッチの解析、スケジューリング、キャンセル、セッションの call/result ログを所有します。この図は、共有された呼び出しパイプラインが、ループを変えずに、ポリシー、フック、サンドボックス、ファイルシステム・ガード、結果の書き換え、最終結果の観測、UI レンダリングをどこで適用するかを示します。最初に `tools/pre-execute` ウォーターフォールが走り、次に単調ガード、その後に `tools/execute` と `tools/post-execute` のウォーターフォールが続きます。3つのウォーターフォールは呼び出しを変換できます。定義が所有する `projectContent` は post-execute の前に準備済みコンテンツを設定し、`finalizeContent` と `tools/result` はその後に走ります。

```mermaid
flowchart TD
  model["Assistant message contains tool-call block"]
  toolCall["Session event: <code>tool/call</code><br/>logged before execution"]
  presentCall["UI pending card<br/>presentCall(args)"]
  pre["<code>tools/pre-execute</code> waterfall<br/>hooks, permission, sandbox"]
  guards["Registered monotonic guards<br/>deny or abstain; identity protected"]
  denied["denied or approval refused<br/>tool body skipped"]
  approval["<code>ctx.approval</code> one-shot prompt<br/>absent or unanswerable: deny"]
  around["<code>tools/execute</code> waterfall<br/>timeout, retry, metrics (around dispatch)"]
  toolBody["Registered tool execute() body"]
  fsGate["<code>fs/write-intent</code> or <code>fs/edit-intent</code><br/>tool-fs mutations only"]
  owned["Tool-owned session events<br/><code>todo/write</code>, <code>fs/observed</code>, <code>hook/invoked</code>, <code>hook/result</code>, <code>tool/ptc-dispatch</code>"]
  project["ToolDefinition.projectContent<br/>execution-prepared text and images"]
  post["<code>tools/post-execute</code> waterfall<br/>accept, block, replace, add context"]
  normalized["Invocation pipeline normalization<br/>pipeline/result snapshot throws become isError"]
  finalize["ToolDefinition.finalizeContent<br/>last content-only invariant"]
  final["<code>tools/result</code> synchronous notification<br/>frozen authoritative outcome"]
  context["Active-batch additionalContexts FIFO<br/>injected user/message after recorded tool results"]
  toolResult["Session event: <code>tool/result</code><br/>single model-facing outcome"]
  allResults["Tool batch settled<br/>recorded tool/result events complete"]
  presentResult["UI completed card<br/>presentResult(args, result)"]
  executor["Selected ToolExecutor.execute<br/>raw batch, Agent, optional logging step"]
  model --> executor --> toolCall
  toolCall --> presentCall
  toolCall --> pre
  pre -->|allow| guards
  guards -->|allow| around
  guards -->|deny| denied
  guards -.->|throw| normalized
  around --> toolBody
  pre -->|deny| denied
  pre -->|ask| approval
  approval -->|allowed-once| guards
  approval -->|rejected, cancelled, unavailable| denied
  approval -.->|throw| normalized
  denied --> project
  pre -.->|throw| normalized
  toolBody --> fsGate
  fsGate --> toolBody
  toolBody --> owned
  toolBody --> around
  around --> project
  project --> post
  project -.->|throw| normalized
  around -.->|wrapper throws| normalized
  post -.->|throw| normalized
  post --> finalize
  normalized --> finalize
  finalize --> final
  final --> toolResult
  toolResult --> presentResult
  toolResult --> allResults
  allResults --> context
```

ファイルシステムの read-before-edit チェックは、`fs/*` イベント上の `tool-fs` より下に留まります。汎用の pre/post ウォーターフォールはフックと承認ポリシーをホストします。`ctx.approval` は単調ガードの前に ask を解決し、並べ替えてはならないオーナー・ポリシーは登録済みガードのままです。タイムアウトのような around ディスパッチの関心事は `tools/execute` をラップします。パイプラインは候補結果をロスレスにスナップショットし、スナップショット失敗を正規化してから、可視定義のスナップショット済み `finalizeContent` コールバックが同期的なコンテンツのみの不変条件を強制します。その後 `tools/result` が、不変でロスレス JSON の結果を観測します。これにより、フックはツールを1つのポリシー・サービスに結合せずに、ツール・ファミリーをまたげます。PTC モードは、予約された `run_code` トランスポートとその順序付きサブコールの両方をパイプラインに通します。サブコールは親トークンを運び、`tool/ptc-dispatch` をログし、拒否をバインディング拒否として返し、call/result の隣接性を保つために `additionalContexts` を省略します。

メンテナンス・モード: 厳選された Mermaid フローです。正確なツール・スキーマとイベント署名は、生成されたカタログにあります。
