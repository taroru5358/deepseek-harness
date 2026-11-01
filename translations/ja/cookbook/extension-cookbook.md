# クックブック: 拡張プラグインの形

日本語 | [English](../../../docs/cookbook/extension-cookbook.md) | [中文](../../../docs/cookbook/extension-cookbook.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cookbook/extension-cookbook.md](../../../docs/cookbook/extension-cookbook.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

harness 拡張のためのリファレンス・パターンです。スニペットは import とヘルパー実装を省略しており、コピー＆ペーストで完結するものではありません。具体的な作成手順は、[パッケージ・チェックリスト](adding-a-package.md)、[最初のツールのチュートリアル](../user/develop/basic/tool.md)、[ツール・リファレンス](adding-a-tool.md)、[LLM アダプター・ガイド](adding-an-llm-adapter.md)、[Session 形式バージョンのチュートリアル](adding-a-session-format-version.md)を参照してください。システムと拡張ポイントのマップは[アーキテクチャ](../architecture.md)が所有します。

## ツール・プラグイン

ツールは `ctx.tools` に登録します。注釈付きの `defineTool` の例（型付き `execute` 引数、結果の構築、`run_in_background` パターン）は [adding-a-tool.md](adding-a-tool.md) にあります。そのガイドがツール定義の正です。生の JSON-Schema `ToolDefinition` も `ctx.tools.register()` に直接受け入れられます（MCP 由来のツールはこうして届きます）。`defineTool` はファーストパーティ・ツール用の型付きヘルパーです。

## フック・プラグイン（パーミッション・ゲートの例）

このパーミッション・ゲートはフック・プラグインの一例です。`tools/pre-execute` ゲートから型付きの決定を返し、呼び出しを許可または拒否します。sandbox、permission、plan-mode のプラグインもこの拡張ポイントを使えます。フック・プラグインは他の拡張ポイントを傍受でき、本質的にパーミッション・ゲートではありません。「ネイティブ・フック」は傍受ポイント上の通常の Cordis プラグインであり、外部プロトコルを必要としません。

```ts
import type { Context } from '@deepseek-ai/cordis'
import type { PreToolDecision, ToolExecution } from '@deepseek-ai/dsh-tools'

declare function isAllowed(exec: ToolExecution): Promise<boolean>

export const name = 'permission-gate'

export function apply(ctx: Context) {
  ctx.on('tools/pre-execute', async (exec, next): Promise<PreToolDecision> => {
    if (!(await isAllowed(exec))) {
      return { kind: 'deny', reason: 'Denied by policy.' }
    }
    return next()
  })
}
```

この waterfall は並べ替え可能なポリシー層です。不変条件が単調な最終拒否を必要とする場合は `ctx.tools.guard()`、プラグインがディスパッチの寿命を包む必要がある場合は `tools/execute`（タイムアウト／リトライ／メトリクス。置き換え可能なのは `exec.signal` だけ）、明示的な結果変換には `tools/post-execute`、不変の最終結果に対する閉じ込められた観察には `tools/result` を使います。選択ルールは [adding-a-tool ガイド](adding-a-tool.md#execution-policy-and-observation)にあります。

## UI プラグイン

UI プラグインは、永続的な `session/event` レコード（Assistant の決着、turn/step 境界、ツール活動）と、ライブのトークン表示用の一時的な `agent/assistant-stream` フレームを組み合わせ、`agent.followup()` / `agent.steer()` で入力を戻します。組み込み Web Client に業務行を提供するブラウザ・プラグインは、代わりに `ConversationNodeDefinition` とキー付き Chat レンダラーを登録します。[Conversation サブシステム・リファレンス](../subsystems/conversation.md)に従ってください。

```ts
import type { Context } from '@deepseek-ai/cordis'
import { brandString } from '@deepseek-ai/dsh-brand'
import { createUserMessage } from '@deepseek-ai/dsh-llm'
import type { SessionId } from '@deepseek-ai/dsh-session'

declare function render(text: string): void
declare function onUserInput(handler: (text: string) => void): void

export const name = 'my-ui'
export const inject = ['agents']

export function apply(ctx: Context) {
  ctx.on('agent/assistant-stream', ({ frame }) => {
    if (frame.type === 'chunk' && frame.chunk.type === 'text-delta') {
      render(frame.chunk.text)
    }
  })
  onUserInput(text => ctx.agents.get(brandString<SessionId>('client-session'))?.followup(createUserMessage({
    content: [{ type: 'text', text }],
    source: { kind: 'user' },
  })))
}
```

## 外部プロトコル・ドライバー

*プロトコル・ドライバー*は、ワイヤー上のピアを `ctx.agents` に適応します。UI にサービスしても、自動化クライアントにサービスしてもかまいません。stdio ドライバーは stdout を所有し、ファクトリーを通じてエージェントを作成または再開し、プロトコル・リクエストを `followup()` または `cancel()` にマップします。低レベルのプロンプト・リクエストは、永続的なエンキュー受領を返します。`MessageId` と `turn/end` を相関させて結果を取得することはしません。エージェント全体のステータスは別途公開します。自動化メソッドは、受領から次の idle まで待機し、その明示的に所有する区間を要約してもかまいません。一方、UI は通常、終わりのないイベント・ストリームを観察し続けます。エージェントは `AgentHandle.dispose()` で破棄し、破棄が静止に到達するようにします。

[`packages/acp/acp`](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/acp/acp) は自動化専用の実例です。Agent Client Protocol JSON-RPC stdio 上で新鮮なテキスト・セッションを公開し、コミット済みの assistant テキストを emit し、所有するエージェント用にワンショットのマシン・パーミッション回答器を登録します。正確なメソッド、イベント順序、ライフサイクル・コントラクトはその [README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/acp/acp/README.md) が定義します。

```ts
import type { Context } from '@deepseek-ai/cordis'
import { expandAssistantStream } from '@deepseek-ai/dsh-llm'

export const name = 'my-protocol-bridge'
export const inject = ['agents', 'sessions', 'sessionPersistence']

export function apply(ctx: Context) {
  // Publish every committed Assistant text delta to the client.
  ctx.on('session/event', (_session, event) => {
    if (event.type === 'assistant/message' || event.type === 'assistant/attempt') {
      for (const { chunk } of expandAssistantStream(event.data.stream)) {
        if (chunk.type === 'text-delta') {
          // sendToClient({ kind: 'message_chunk', text: chunk.text })
        }
      }
    }
  })
  // Inbound "prompt": create/resume an agent, feed it, and return its enqueue receipt.
  // Whole-agent status is a separate notification; no turn end belongs to this prompt.
  // Teardown reaches quiescence via AgentHandle.dispose() (stop + await exit).
}
```

## 実行可能な配線

出荷済みアプリケーションは `packages/bundle/*/cordis.patch.yml` を通じてプロファイル・レイヤーを提供し、製品の `dsh` ランチャーが名前付きプロファイルを通じて Web、ACP、SDK、ワンショット・ヘッドレス実行を所有します。任意のユーザー向けオーバーレイは `apps/cli/config/examples/` にあり、プロファイル統合テストは `apps/cli/tests/profiles/` にあります。パッケージ固有の Loader 構成は、そのパッケージのテストのそばに留まります。

## 機能 → メカニズムのマップ

すべての製品機能は、文書化された拡張ポイント上のリスナーにマップされます。これはマイクロカーネルの主張を検証可能にしたものです（[マイクロカーネル・リファレンス](../architecture.md)）。ループを変更する行はありません。

`system-prompt/assemble` はエキスパート向けの協調的なアセンブリ全体変換です。返されたアセンブリが権威となるため、リスナー作者はアクティブな PTC モードと structured-output プロトコルの貢献を保持する責任を負います。プレゼンテーション、lookup、実行を通じて整合させ続ける必要があるツール・フィルタリングには、`ctx.tools.restrict()` を優先してください。

| 製品機能 | プラグイン・メカニズム |
|---|---|
| フック・システム（ユーザー + プロジェクト・レベル） | `agent/created`、`agent/pre-step`、`tools/pre-execute`、`tools/post-execute`、`agent/turn-stopping` 上のリスナー。waterfall は型付きの決定を返し、`agent/turn-stopping` は別の step を steer してもよい。`dsh-hooks-claude-code` / `dsh-hooks-codex` ブリッジがフック設定ファイルをこれらの拡張ポイントにマップする |
| `/goal` | `ctx.goals` が永続状態を所有し、`dsh-goal-round-driver` が公開 `Agent` を通じて同一セッションのラウンドをスケジュールし、別々のコマンド／ツール・プロデューサーが人間／モデルの制御を公開する |
| `/loop` | `turn/end` セッション・イベントで次の反復を `followup()` する。または強制的に継続する |
| 動的ワークフロー | `ctx.workflowEngine` + PTC ワークフロー・エンジン + `workflow` ツール。構造化されたインプロセス子が、スコープ付きプロンプト／ツール登録、単調なツール・ガード、最終 `tools/result` コミット（囲む `run_code` を含む）、structured-output 実行の単調な `concludeTurn()` マーカーで出力を強制する |
| キュー済み + steering メッセージ | コアの `Agent.followup()` / `Agent.steer()` |
| コンテキスト・コンパクション（自動 + 手動） | `ctx.compaction` シーム + `dsh-compaction-basic`。自動プレッシャーは直列の `agent/pre-step` で実行され、正規のオーバーフロー復旧は `agent/request-error` で実行され、手動の呼び出し元は同じ compact サービスを使う（[compaction リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction/README.md)） |
| システム・プロンプトの設定可能性 | 順序付けとスコープ・ローカル・シャドウイングを伴う `ctx.systemPrompt.section()` |
| AGENTS.md（ルート） | ファイルを読むセクション・プロバイダー |
| AGENTS.md（サブディレクトリ、on-touch）+ ファイル変更通知 | watcher / tool-result リスナーからの `agent.inject()` |
| 組み込みツール | `ctx.tools.register()`。スキーマは自動的にアセンブリへ流れ込む。`dsh-tool-*` ファミリー（bash、fs、web、subagent、todo）が出荷済みの例 |
| ToolSearch / 漸進的開示 | 可視セットの変化に応じてスコープ付き `ctx.tools.restrict()` 登録を置き換える。レジストリがプレゼンテーション、lookup、実行を整合させる |
| ツールの期限 / リトライ / メトリクス | コア・ディスパッチを `tools/execute` で包む。ラッパーは1つのレキシカルな寿命の中で `exec.signal` を置き換え、委譲し、正規化された結果を検査できる |
| 最終ツール結果のメトリクス / 監査 / キャプチャ | 不変の権威ある結果を `tools/result` で観察する。プラグインが結果を変換したりコンテキストを付加したりする必要がある場合にだけ、代わりに `tools/post-execute` を使う |
| 単調な終端 turn ポリシー | 成功した終端ツールから `ToolExecution.concludeTurn()` を呼ぶ。同じレスポンス内の後続ツール呼び出しは引き続きガード可能で、ループは step の後に停止する |
| サブプロセス sandbox（landlock / sandbox-exec） | `dsh-bash-sandbox` を通じて `ctx.sandbox` バックエンドを使う。ケイパビリティ・レベルの拒否には `tools/pre-execute` を使う |
| パーミッション・システム / AskUserQuestion | `tools/pre-execute` から `ask` を返し、`ctx.approval` を通じて回答する。通常のユーザー質問には、別のモデル向け ask ツールを登録する |
| Plan モード | [`@deepseek-ai/dsh-plan-mode`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/plan/plan-mode/README.md) — 記録される `plan/mode` 状態、`plan:policy` ガイダンス・セクション、`/plan [message]` エントリー、`/plan off` の直接終了、ユーザーがレビューする `exit_plan_mode` 終了。強制は独立した sandbox／承認の軸に留まる |
| サブエージェント委譲 | `ctx.subagents` プロバイダー・レジストリ（`dsh-subagent-spawn-in-process`/`dsh-subagent-fork-in-process`/`dsh-subagent-acp`/`dsh-subagent-codex`/`dsh-subagent-claude-code`/`dsh-subagent-dsh-sdk`）+ `dsh-tool-subagent` が、設定された1つのプロバイダーをモデルに公開する |
| MCP | サーバーごとに1つのプラグイン: ツールを発見 → `ctx.tools.register()` |
| スキル | セクション + ツール登録。呼び出し時にスキル内容を `inject()` する |
| メモリ | セクション・プロバイダー + ツール |
| スケジュール済みタスク（cron） | プラグインがモデルから呼び出せるスケジューリング・ツールを登録する。期限が来たタイマーは `followup(…, {source: {kind: 'schedule'}})` を通じて配信する |
| UI（GUI。CLI は JSONL を emit） | ライブ・チャンクには `agent/assistant-stream`、永続的な決着、境界、ツール活動には `session/event` をリッスンする。入力 → `followup()` |
| Web Client Chat の業務ノード | `ConversationNodeDefinition` と `conversation.chat.node` キー付きレンダラーを登録する |
| SessionTelemetryBackend / リプレイ可能なトレース | `session/event` → JSONL。リプレイ = `sessions.create(id, { seed })` |
| モデル・アダプター | `registerAdapter` 経由の `LlmAdapter` サブクラス（`dsh-llm-deepseek`、`dsh-llm-pi-ai`） |
| プラグイン・ホットリロード | すべての登録は `ctx.effect` → vendored HMR がそのまま動く |
