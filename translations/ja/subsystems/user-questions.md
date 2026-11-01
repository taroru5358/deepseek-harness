# ユーザー・インタラクション

日本語 | [English](../../../docs/subsystems/user-questions.md) | [中文](../../../docs/subsystems/user-questions.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/user-questions.md](../../../docs/subsystems/user-questions.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[dsh-user-questions](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/interaction/user-questions) の user-questions seam。agent が続行する前に人間の回答を必要とするとき、ツールまたはパーミッション・プラグインが使うプロバイダー中立の語彙である。agent スコープの waterfall リスナーが利用可能な UI サーフェスを構成し、接続済みクライアントへ中継されるリスナーも含む。

ソース: [`packages/interaction/user-questions/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/interaction/user-questions/src/index.ts)

## 質問オプション

`AskUserQuestionOption` は 1 つの選択可能な選択肢を含む。`label` はユーザー向けのオプション・テキストであり、同時にモデル向けの選択値でもある。`description` は任意の UI ヘルプ・テキストである。

```ts type-equiv
/** One selectable answer offered to the user. */
interface AskUserQuestionOption {
  /** User-facing label. */
  label: string
  /** Optional extra context rendered by capable UIs. */
  description?: string
}
```

## プレゼンテーション意図

`AskUserQuestionIntent` はオプション・ラベルと呼び出し元の決定ポリシーを保ったままプレゼンテーションを変える。`plan-review` は承認オプションを名指し、plan の詳細を要求する。`approval` は詳細付きの 1 つのブロッキング質問と、提示されたオプションを正確にカバーする、相異なる reject、approve、任意の always-allow ラベルを要求する。マルチセレクトと時間制限付き待機は拒否する。CUA フロントエンドは既存のコンポーザー・チェーンを通じて固定の承認を描画し、通常の質問と plan レビューは既存のレンダラーを保つ。

```ts type-equiv
/**
 * A caller-declared presentation intent: the question IS this kind of
 * decision, so a UI that recognises the tag may present it as such instead of as a
 * generic option list. Tagged so further intents can be added; a UI that does
 * not know a tag renders the generic flow, and the answer encoding is identical
 * either way — an intent changes presentation only, never the protocol.
 */
type AskUserQuestionIntent = {
  /** A plan submitted for review: `detail` is the plan markdown `ask()` requires, and the decision approves or declines it. */
  kind: 'plan-review'
  /**
   * The option label that approves the plan; every other option declines it.
   * Named rather than positional so no UI infers the verdict from option order.
   * An `approve` naming no option of its own question is rejected at `ask()`.
   */
  approve: string
  /** Logged tool invocation whose arguments contain the reviewed plan. */
  callId?: ToolCallId
} | {
  /** A blocking permission decision with fixed actions and no free-text answer. */
  kind: 'approval'
  /** Option label granting the scope described by the caller. */
  approve: string
  /** Option label refusing the requested access. */
  reject: string
  /** Optional persistent grant offered only when the caller supports it. */
  always?: string
  /** Metadata for human recognition; the caller admits icon sources, which the UI may load as URLs. Never an authorization identity. */
  subject?: { name: string; icon?: string }
  /** Localized question and limited-grant label; fixed cards own reject/always wording. Option labels remain answer values. */
  display?: {
    question: { en: string; [locale: string]: string }
    detail?: { en: string; [locale: string]: string }
    approve?: { en: string; [locale: string]: string }
  }
}
```

## 質問項目

`AskUserQuestionItem` はリクエスト内の 1 つの質問である。呼び出し元は安定した `id` を供給し、それが回答とともにエコー・バックされるため、バッチ化された質問もルーティング可能なままである。任意の `detail` は補足テキストを運び、プロバイダーはそれを質問とともに描画するが、選択可能なオプション・ラベルには含めない。

```ts type-equiv
/** One question in a user-questions request. */
interface AskUserQuestionItem {
  /** Stable caller-provided question id, echoed in the answer. */
  id: string
  /** The question to display. */
  question: string
  /** Optional supporting detail rendered with the question but kept out of option labels. */
  detail?: string
  /** Optional short heading/group label. */
  header?: string
  /** Optional choices the UI can render as a menu. */
  options?: AskUserQuestionOption[]
  /** Whether more than one option may be selected. Defaults to single-select. */
  multiSelect?: boolean
  /** Optional presentation intent for capable UIs; absent asks for the generic option list. */
  intent?: AskUserQuestionIntent
}
```

## 質問リクエスト

`AskUserQuestionRequest` はパッケージをまたぐリクエストである。`questions` が配列なのは、UI が関連する prompt を 1 つのフローで提示しつつ、回答ごとに安定した id を保てるようにするためである。存在する場合、`agent` は正確なライブ呼び出し元であり、interaction seam はライブ・レジストリがそのインスタンスを runtime ルートとして識別する間だけそれを受け入れる。

```ts type-equiv
/** Request for a human answer. */
interface AskUserQuestionRequest extends AskUserQuestionRequestEvent {}
```

## 回答

プロバイダーは質問 id ごとに 1 つの回答項目を返す。`selected` は選択されたオプション・ラベルを含み、`custom` はユーザーが入力した場合の自由形式の「その他」回答を運ぶ。単一選択の質問では、`custom` が選択された選択肢を上書きし、`selected` は空である。複数選択の質問では、`custom` が `selected` のラベルを補足してよい。UI は、それ以外は完了したバッチの中でスキップされた質問を保持するために、`selected` が空で `custom` もない項目を使うこともできる。

```ts type-equiv
/** Answer to one question. */
interface AskUserQuestionAnswerItem {
  /** The answered question id. */
  id: string
  /** Selected option labels. May accompany custom text for a multi-select question. */
  selected: string[]
  /** Optional free-text "Other" answer. */
  custom?: string
}
```

```ts type-equiv
/** The human's answer. */
interface AskUserQuestionAnswer {
  /** Structured answers keyed by question id. */
  answers: AskUserQuestionAnswerItem[]
}
```

時間制限付き回答者は `wait: { callId, timed: true }` を受け取る。call id はカードとフォアグラウンド待機を識別する。Client は `attachWait` を呼び出して待機を要求し、残り時間を取得する。トランスポートはこれらのフィールドを検査も書き換えもせず、元のイベントを転送する。`timed` のない `wait` は、カード・キー付きの無期限リクエストを識別する。`wait` がないことは、カード・キーのないブロッキング・リクエストを識別する。

```ts type-equiv
/** Client-safe payload declared for the user-question answerer waterfall. */
interface AskUserQuestionRequestEvent {
  /** Questions to display. */
  questions: AskUserQuestionItem[]
  /** Agent identity projected to the corresponding Client Context in transit. */
  agent?: Agent
  /** Cancellation lifetime of the pending request. */
  signal?: AbortSignal
  /**
   * Tool call the Client card is keyed by. Timed answerers attach to the
   * business wait stream before starting their local countdown.
   */
  wait?: {
    /** Tool call the Client card is keyed by. */
    callId: ToolCallId
    /** True for a foreground wait that requires a Client claim; absent for indefinite waits. */
    timed?: boolean
  }
}
```

## 時間制限付き質問と遅延返信

時間制限付き質問は、境界のあるフォアグラウンド・ウィンドウを待ってから、agent が独立した作業を続けられるようにする。ユーザーがウィンドウ内で回答すれば、ツールは回答バッチを返す。そうでなければ `TimedUserQuestionResult` は `{ pending: true, callId }` である。pending は質問がまだ回答可能であることを意味し、空の回答、拒否、確認、パーミッションではない。

```ts type-equiv
/** Timed ask result returned when the foreground answer window closes. */
type TimedUserQuestionResult = AskUserQuestionAnswer | { pending: true; callId: ToolCallId }
```

Client の要求は Gateway 配信ではなく、ビジネス Remote ストリームである。`TimedQuestionWait` は元の Host 期限を保持し、要求が存在しない間だけタイマーを動かす。その `attachWait` ストリームは現在の残り時間を yield し、Client は自身の時計でカウントダウンする。フォーカス、編集、Take time はローカルに留まる。ストリームのキャンセルはその要求を解放し、最後の要求が去った後、Host は元の期限を再開する。無人のタイムアウトはフォアグラウンド waterfall のプライベート・シグナルを abort し、Turn をキャンセルせずに pending を返す。Gateway と `api-remotes` は質問固有のタイミング・ポリシーを含まない。

`userQuestions` Session projection は、既存のイベントを回答可能な質問セットと決着済みセットに畳み込む。待機状態、フォーカス状態、期限はログに記録されない。各 `request/header` はモデルが見た正確なツール・スキーマを記録する。ネイティブ呼び出しは、そのヘッダーが `timeout` パラメーター付きの時間制限付き `ask_user_question` スキーマを宣言している場合にのみ追跡される。ブロッキングのレガシー・ツールは決してそれを宣言しない。ネイティブの `tool/call` が質問を開く。`tool/result` は、pending ペイロードを運ぶ場合、または Session 再開修復が追加する合成の `TOOL_OUTCOME_UNKNOWN` 結果である場合にのみ、その質問を `continued` にする。回答バッチはそのバッチとともに `settled` へ移し、あらゆる失敗はそれを落とす。`run_code` の PTC サブコールは、リクエスト・ヘッダーが `run_code` しか公開しないにもかかわらず、`tool/ptc-dispatch` の結果が pending のときに projection に入る。遅延返信は、そのメッセージが `user/message` として受け入れられたときに、そのメッセージから読み出した回答バッチとともに、continued の質問を `settled` へ移す。時間制限付き呼び出し自身の結果がタイムアウトを記録しているため、transcript 行にはユーザーが最終的に何を選んだかを読む他の場所がない。キューに入った返信は受け入れ前にキャンセルでき、質問は回答可能なまま残る。テキストに読み取り可能なバッチを運ばない返信は、回答なしで呼び出しを決着させる。アクティブでない呼び出しを名指す返信は何も変えない。

```ts type-equiv
/** `open` while the tool call may still return the answer; `continued` once the answer can only arrive as a new turn. */
type UserQuestionState = 'open' | 'continued'
```

```ts type-equiv
/** One unanswered timed `ask_user_question` call reconstructed from the Session log. */
interface PendingUserQuestion {
  readonly callId: ToolCallId
  readonly questions: readonly AskUserQuestionItem[]
  readonly state: UserQuestionState
}
```

```ts type-equiv
/**
 * One timed `ask_user_question` call and the answers it settled with: the
 * batch its own result carried when the user answered inside the window,
 * otherwise the batch its late reply carried, because that call's own result
 * recorded the timeout. A transcript row reads what the user finally answered
 * from here, and only a call listed here was a timed one.
 */
interface SettledUserQuestion {
  /** The settled call. */
  readonly callId: ToolCallId
  /** The recorded batch, one entry per question; empty when the late reply carried none. */
  readonly answers: readonly AskUserQuestionAnswerItem[]
}
```

```ts type-equiv
/**
 * Both halves of one Session's timed `ask_user_question` state, as every
 * Client reads them. A call made while the blocking legacy tool was in
 * effect appears in neither half.
 */
interface UserQuestionProjectionView {
  /** Calls that can still take an answer, in ask order. */
  readonly active: readonly PendingUserQuestion[]
  /** Calls an answer settled, in settlement order. */
  readonly settled: readonly SettledUserQuestion[]
}
```

オープンな質問は waterfall を通じてのみ回答を受け付ける。`continued` になると、`answer` Remote メソッドが所有 agent 向けのユーザー・メッセージをキューに入れ、アイドルなら起こす。メッセージのソースは `user-question-reply`、結果は `answered`、ペイロードは `answer_to_pending_question` である。このメソッドはオープンまたは未知の呼び出しに対して `false` を返し、Host 再起動後であっても 2 番目のキュー済み返信を `REPLY_QUEUED` で拒否し、不完全または重複した回答バッチを `BAD_ANSWER` で拒否する。

Client パネルを閉じても何も送信されない。フォアグラウンド・タイムアウトはツールの待機だけを終わらせる。continued の質問には期限がなく、返信が受け入れられるまで回答可能なままである。

Host 再起動後、Remote 層は Session が再び開かれたときにルート agent を再開する。その後、遅延返信は新しい turn に入る。

## エラー

`UserQuestionError` は `HarnessError` を継承するため、`ctx.tools.execute()` は `EMPTY_QUESTIONS`、`BAD_INTENT`、`NO_PROVIDER`、`ASK_ABORTED`、UI 側キャンセルといったモデル向けツール失敗の `{ name, code }` を保持する。

```ts type-equiv
/** Stable error taxonomy for user-questions failures. */
class UserQuestionError extends HarnessError {
  constructor(message: string, code: string, options?: ErrorOptions) {
    super(message, code, options)
    this.name = 'UserQuestionError'
  }
}
```

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxuserquestions--userquestionservice"></a>

### `ctx.userQuestions` — `UserQuestionService`

`ctx.userQuestions`: 検証と、スコープ付き回答者 waterfall。

```ts cordis-catalog
/**
 * Answer a continued question. The reply is steered into the agent as a
 * user message whose source names the call; that message is also the
 * record that closes the question in the projection.
 * @param agent - Live root agent for the owning Session.
 * @param callId - Continued question identity.
 * @param answer - Complete structured answer batch, one item per question of the call.
 * @returns Whether the question is still continued; an accepted reply stays
 *   queued until the agent admits its user message.
 * @throws {UserQuestionError} `BAD_ANSWER` when the batch does not name each
 *   question of the call exactly once, or `REPLY_QUEUED` when a reply is
 *   already waiting for admission.
 */
@Remote answer(agent: Agent, callId: ToolCallId, answer: AskUserQuestionAnswer): boolean

/**
 * Let one answer UI hold a live timed wait. Closing the stream releases its claim.
 * @param agent - Live root agent owning the question.
 * @param callId - Foreground tool call to attach to.
 * @param signal - Remote stream cancellation, including Client disconnect.
 * @returns One Host-computed remaining duration, or no frames once the wait ended.
 */
@Remote({ mode: 'stream' }) async *attachWait(agent: Agent, callId: ToolCallId, signal: AbortSignal): AsyncIterable<{ remainingMs: number }>

/**
 * Foreground wait whose first settlement the Client decides: the Client
 * rejects with `ASK_TIMED_OUT` when its countdown ends, and this method maps
 * that code to the pending result.
 * @param request - Questions, live owner agent, and abort signal.
 * @param callId - Tool call identity the Client card is keyed by.
 * @param timeoutMs - Positive foreground wait in milliseconds.
 * @returns The answer when it arrives inside the window, otherwise a pending
 *   result, also when no connected Client claimed the request by the deadline.
 * @throws {UserQuestionError} `BAD_TIMEOUT` for a non-integer, non-positive,
 *   or oversized wait.
 */
async askTimed( request: AskUserQuestionRequest & { agent: Agent }, callId: ToolCallId, timeoutMs: number, ): Promise<TimedUserQuestionResult>

/**
 * Ask the scoped answerer waterfall and wait for the user's answer.
 *
 * When a caller supplies an agent, human interaction is valid only for the
 * exact live runtime root. Runtime ownership, not durable session lineage,
 * decides this boundary: an owned child has no human answerer and would
 * block forever, while a lineage-bearing session resumed as a new runtime
 * root may ask normally.
 *
 * @param request Questions, owner agent, and abort signal.
 * @returns The answer chosen or typed by the human.
 * @throws {UserQuestionError} code `ASK_ABORTED` when the supplied signal
 *   is already or becomes aborted, `CALLER_NOT_LIVE` when a supplied agent
 *   is not the registry's exact live instance, or `DELEGATED_CALLER` when
 *   that live agent is owned by another agent.
 */
async ask(request: AskUserQuestionRequest): Promise<AskUserQuestionAnswer>
```

型: [Agent](core.md) · [ToolCallId](core.md)

ソース: [`packages/interaction/user-questions/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/interaction/user-questions/src/index.ts)

<a id="user-questions-events"></a>

### `user-questions/*` イベント

<a id="user-questionsrequest--waterfall"></a>

#### `user-questions/request` — waterfall

構成済みの回答者に構造化されたユーザー入力の提供を求める。回答を返してリクエストを要求するか、`next()` を呼んで委譲する。スコープ・フィルター付きディスパッチ（`@deepseek-ai/dsh-scope`）: agent スコープのリスナーはその agent だけを受け取る。

```ts cordis-catalog
/**
 * Ask composed answerers for structured user input. Return an answer to
 * claim the request or call `next()` to delegate. Scope-filtered dispatch
 * (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent.
 * @param request - pending user-question request.
 * @mode waterfall
 */
'user-questions/request'( this: Scoped<Agent>, request: AskUserQuestionRequestEvent, next: () => Promise<AskUserQuestionAnswer>, ): Promise<AskUserQuestionAnswer>
```

型: [Agent](core.md) · [Scoped](scope.md)

ソース: [`packages/interaction/user-questions/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/interaction/user-questions/src/types.ts)
<!-- END GENERATED cordis-surface -->
