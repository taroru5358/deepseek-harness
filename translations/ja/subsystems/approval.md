# ユーザー承認

日本語 | [English](../../../docs/subsystems/approval.md) | [中文](../../../docs/subsystems/approval.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/approval.md](../../../docs/subsystems/approval.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[dsh-user-approval](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/interaction/user-approval) のユーザー承認 seam は 1 つの問いに答える。この特定のアクションを続行してよいか？ 共有のリクエスト/結果語彙、`ctx.approval` dispatch サービス、`approval/request` 回答者ウォーターフォール、ログ専用の監査ペア、セッションごとの `ask`/`never` ポリシーを所有する。UI チャネルは人間の回答者を提供できる。[ACP 自動化ブリッジ](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/acp/acp)は自身の agent にワンショットのマシン判定を提供する。[dsh-tools](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/core/tools) や [dsh-tool-bash](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/shell/tool-bash) のような呼び出し元は閉じた結果を消費し、`allowed-once` でなければ fail-closed する。

ソース: [`packages/interaction/user-approval/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/interaction/user-approval/src/index.ts)

## アイデンティティと結果

すべてのリクエストは新しい `ApprovalRequestId` を受け取る。このブランドは `approval/asked` と `approval/decided` の監査イベントを対にする一方で、承認 id をツール呼び出しや agent/session の id と交換可能にはしない。

```ts type-equiv
/**
 * Pairs one `approval/asked` audit event with its `approval/decided`.
 * Service-issued (one fresh id per {@link ApprovalService.request} call).
 */
type ApprovalRequestId = Branded<'ApprovalRequestId'>
```

`ApprovalOutcome` は閉じており fail-closed である。`allowed-once` は問い合わせ対象のアクションだけを許可する。呼び出し元は `rejected`、`cancelled`、`unavailable` で拒否する。回答者が存在しない、所有していない、例外を投げる、または規約に適合しない場合は、ゲートを開くのではなく `unavailable` になる。

```ts type-equiv
/**
 * Closed approval outcomes: a one-shot grant, explicit rejection, withdrawn
 * request, or unavailable answerer. Callers fail closed on `unavailable`.
 */
type ApprovalOutcome = 'allowed-once' | 'rejected' | 'cancelled' | 'unavailable'
```

## セッションごとのポリシー

`ApprovalPolicy` は、対話型回答者が実行される前に何が起こるかを決める。`ask` は合成された回答者チェーンに委譲し、回答がない場合の既定は `unavailable` である。`never` は回答者を一切 dispatch せずに決定的に `rejected` を返す。有効な値はセッション・ログの最後の `approval/policy` イベントであり、なければサービス設定にフォールバックする。コンシューマーは `ctx.approval.effectivePolicy(session)` で読み取る。`setApprovalPolicy(session, policy)` が唯一の書き込み経路なので、リプレイはオーバーライドを再構築する。

```ts type-equiv
/**
 * A session's approval policy — what happens to an {@link ApprovalService}
 * ask BEFORE any interactive answerer sees it:
 *
 * - `'ask'` (the default) — delegate to the composed answerers; with none
 *   composed the chain falls through to the fail-closed `'unavailable'`.
 * - `'never'` — never prompt anyone: every ask resolves `'rejected'`
 *   deterministically. The strict headless stance (CI, unattended runs) and
 *   the policy whose outcome is knowable without asking.
 */
type ApprovalPolicy = 'ask' | 'never'
```

どちらのポリシーも、その完全な現在の意味をキャッシュ安全なランタイム・コンテキスト・スナップショットに寄与する。ソース付きの `user/message` が永続的なモデル可視入力である。承認状態を変更すると、レンダリング済みシステム・プロンプトを保持する `system/message` ノードに触れずに、保持履歴の後に新しい完全スナップショットを追記する。

## 承認リクエスト

`ApprovalRequest` は、問いをルーティングして監査するのに十分な精度で agent とツール・アクションを識別する。ツール引数は意図的に省略する。回答者は `callId` を通じて既にストリームされたツール呼び出しにプロンプトを添付し、ずれる可能性のある 2 つ目のコピーをレンダリングしない。任意の `displayReason` はリクエスター所有のロケール文字列をプレゼンテーションに運ぶ。`reason` は監査テキストのままである。

```ts type-equiv
/**
 * Readonly same-process permission question. `callId` links to an already
 * presented tool call, so arguments are not duplicated here.
 */
interface ApprovalRequest extends ApprovalRequestEvent {
  /**
   * The agent on whose behalf the question is asked. Routes the question (a
   * UI answerer only answers for agents it owns) and receives the audit
   * events on its session log.
   */
  readonly agent: Agent
  /** The tool the question is about (presentation and audit). */
  readonly toolName: string
  /**
   * The exact tool call being decided, when the asker has one — lets a UI
   * attach the prompt to the tool call it already streamed.
   */
  readonly callId?: ToolCallId
  /** The asker's human-readable explanation of WHY it is asking. */
  readonly reason?: string
  /**
   * Aborting withdraws the question: the request settles `'cancelled'`
   * immediately and a late answer from a still-pending answerer is discarded.
   */
  readonly signal?: AbortSignal
}
```

## ディスパッチと監査

`ctx.approval.request(req)` は、要求元のセッションが開いている turn の内側にあることを必要とする。`approval/asked` を追記し、1 つの結果を取得し、対応する `approval/decided` を追記して、その結果で解決する。`never` ポリシーはウォーターフォール dispatch の前にサービス内部で強制されるので、後から `prepend` で登録された回答者でもこれを迂回できない。回答者は、リクエストを所有する場合は結果を返し、委譲する場合は `next()` を呼ぶ。最初の回答が唯一の決定スロットを占める。

監査イベントはログ専用で、モデルのトランスクリプトには入らない。モデルから見える挙動は、呼び出し元が導出したツール結果と現在のランタイム・コンテキスト・スナップショットである。サービスの破棄はそのコンテキスト・コントリビューションを削除する。回答者リスナーは、所有するプラグインに独立して effect 束縛される。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。dispatch モードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxapproval--approvalservice"></a>

### `ctx.approval` — `ApprovalService`

回答者の前にセッション・ポリシーを適用し、すべての ask/結果ペアを要求元セッションに記録する承認サービス。決定論的なポリシー変更を、ランタイム・コンテキスト・スナップショットと切り替え通知を通じてモデルに公開する。

```ts cordis-catalog
/**
 * Switch one live agent's policy and queue the transition for its next model
 * step. Session initialization uses {@link setApprovalPolicy} directly
 * because there is no previously visible policy to change.
 * @param agent - the live agent whose policy is changing.
 * @param policy - the new effective policy.
 */
setPolicy(agent: Agent, policy: ApprovalPolicy): void

/**
 * Ask the composed answerers to decide one readonly same-process request.
 * The service borrows the request, agent, session, and live signal directly.
 * The request requires an open turn because the audit pair must be enclosed
 * by the durable log's commit/replay boundary; an idle ask rejects before
 * appending anything. The answerer phase always produces an outcome: an
 * aborted signal yields `'cancelled'`, a missing or throwing answerer yields
 * `'unavailable'` (fail closed), and a rogue non-vocabulary return value is
 * normalized to `'unavailable'`. A failure that prevents either audit append
 * from committing still rejects because returning an unlogged decision would
 * violate the pair. Session contains post-commit observer failures, so an
 * authoritative append cannot reject the request or suppress its matching
 * audit event.
 * @param req - the pending decision (agent, tool identity, reason, signal).
 * @returns the closed outcome; `'allowed-once'` is the only grant.
 * @throws when no turn is open or either audit event fails before the session
 *   append commit point.
 */
async request(req: ApprovalRequest): Promise<ApprovalOutcome>

/**
 * Read the session override without applying the configured default.
 * @param session - session whose log supplies the override.
 * @returns the last logged policy, or `undefined` without one.
 */
overrideOf(session: Session): ApprovalPolicy | undefined
```

型: [Agent](core.md) · [Session](session.md)

ソース: [`packages/interaction/user-approval/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/interaction/user-approval/src/index.ts)

<a id="approval-events"></a>

### `approval/*` イベント

<a id="approvalrequest--waterfall"></a>

#### `approval/request` — waterfall

合成された回答者に 1 つの決定を求める。リクエストを引き受ける場合は結果を返し、委譲する場合は `next()` を呼ぶ。スコープ・フィルター付き dispatch（`@deepseek-ai/dsh-scope`）。agent スコープのリスナーはその agent だけを受け取る。

```ts cordis-catalog
/**
 * Ask composed answerers for one decision. Return an outcome to claim the
 * request or call `next()` to delegate. Scope-filtered dispatch
 * (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent.
 * @param req - pending approval request.
 * @mode waterfall
 */
'approval/request'( this: Scoped<Agent>, req: ApprovalRequestEvent, next: () => Promise<ApprovalOutcome>, ): Promise<ApprovalOutcome>
```

型: [Agent](core.md) · [Scoped](scope.md)

ソース: [`packages/interaction/user-approval/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/interaction/user-approval/src/types.ts)
<!-- END GENERATED cordis-surface -->
