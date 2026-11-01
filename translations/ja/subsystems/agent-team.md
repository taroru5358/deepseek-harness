# Agent チーム

日本語 | [English](../../../docs/subsystems/agent-team.md) | [中文](../../../docs/subsystems/agent-team.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/agent-team.md](../../../docs/subsystems/agent-team.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

実験的な暗黙ルート Team ドメイン、モデルツール、host アダプターで共有される型。[Agent Teams Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/feature/2026-08-05-agent-teams.md) は同一性、メールボックス、タスク、共有チェックアウトの決定を所有する。このページは [`packages/experimental/agent-team/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/agent-team/src/types.ts) からの永続的でクライアント可視の形式を記録する。

## 同一性とロスター

`TeamId` は別個の[ブランド](core.md#branded-ids)の下のルート `SessionId` である。`TeamTaskId` は `task-<n>` として割り当てられる。新しい送信はターゲットの受信トレイ `MessageId` を返す。`TeamMessageId` は歴史的なメールボックスレコードのためにのみ保持される。チームメイト名は不変のラベルである。

```ts type-equiv
/** Whole durable value written on every teammate lifecycle change. */
interface TeamMemberSnapshot {
  readonly id: SessionId
  readonly name: string
  readonly description: string
  readonly provider: string
  readonly context: 'fresh' | 'fork'
  readonly phase: TeamMemberPhase
  readonly error?: string
}
```

すべてのメンバーは `provisioning` で開始し、ちょうど 1 つの終端ロスターフェーズ `active` または `failed` に到達する。ロスターの `running`/`inactive` ステータスは別に導出され、このレコードを決して書き換えない。

## メッセージと歴史的メールボックス

新しい送信は受信トレイの受け入れまたは拒否を返す。ターゲットは既存の `agent-message` ソースを、実際の送信者 Session id と送信者名のコンテンツプレフィックスとともに保存する。Team には新しいメッセージのアウトボックスも再送重複排除もない。パッケージ[契約](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/agent-team/README.md#understand-the-implementation)が永続化を定義する。以下のメールボックス型は、歴史的な Lead ログが依然としてリプレイできるように読み取り可能なままである。Team はそれらの保留レコードを配信しない。

```ts type-equiv
/** One peer message recorded by the historical Team mailbox. */
interface TeamMessageSnapshot {
  readonly id: TeamMessageId
  readonly senderId: SessionId
  readonly senderName: string
  readonly targetId: SessionId
  readonly content: ContentBlock[]
}
```

すべてのメッセージは Steer 配信を試みる。実行中のターゲットは最も近いステップ境界でそれを受け取る。非アクティブなターゲットは、ロードされていれば turn を開始し、そうでなければコールドレジュームする。呼び出し元が別のモードを選択できないため、スケジューリングは永続レコードに保存されない。

歴史的な `team-message` ソースは古いメッセージ id と送信者の帰属を保持する。クライアントはそれらをメッセージのタイトルとアイコンに使う。

```ts type-equiv
/** Source recorded by a target Session for one historical Team mailbox delivery. */
interface TeamMessageSource {
  readonly kind: 'team-message'
  readonly teamId: TeamId
  readonly messageId: TeamMessageId
  readonly senderId: SessionId
  readonly senderName: string
}
```

## 共有タスク DAG

すべてのタスクイベントは完全なスナップショットを保存する。`revision` は compare-and-set の値であり、変更ごとに 1 増える。`blockedBy` エッジは削除されていないタスクを名指しし、グラフを非巡回に保たなければならない。`writeScopes` はロックではなく、正規化された助言的なパスプレフィックスである。

```ts type-equiv
/** Whole durable task snapshot; every mutation increments {@link revision}. */
interface TeamTaskSnapshot {
  readonly id: TeamTaskId
  readonly revision: number
  readonly subject: string
  readonly description: string
  readonly status: TeamTaskStatus
  readonly ownerId?: SessionId
  readonly blockedBy: TeamTaskId[]
  readonly writeScopes: string[]
}
```

`pending` は未開始または解放済み、`in_progress` は所有者を持ち、`completed` はブロッカーを満たし、`deleted` は保持されたトゥームストーンである。ビューは所有者名、準備状態、書き込みスコープの重複警告を追加するが、永続スナップショットは変更しない。

<a id="web-projection"></a>

## Web プロジェクション

Lead Session は永続的なロスターローと削除されていないタスクビューを持つ `SessionProjectionMap.agentTeam` を公開する。`failure` は最後の有効な状態の隣に拒否された永続レコードを報告する。メンバーのアクティビティは Session ステータスから来て、モデルラベルは各メンバーの `modelSelection` projection から来る。

```ts type-equiv
/** One durable roster row published through the `agentTeam` Session projection. */
interface TeamMemberProjection {
  readonly id: SessionId
  readonly name: string
  readonly role: 'lead' | 'teammate'
  /** Durable lifecycle; the Lead row is always `active`. Turn activity comes from Session status. */
  readonly phase: TeamMemberPhase
  readonly error?: string
}
```

```ts type-equiv
/** Runtime-enriched task view returned to tools and hosts. */
interface TeamTaskView {
  readonly id: TeamTaskId
  readonly revision: number
  readonly subject: string
  readonly description: string
  readonly status: TeamTaskStatus
  readonly blockedBy: TeamTaskId[]
  readonly writeScopes: string[]
  readonly ownerName?: string
  readonly ready: boolean
  readonly writeScopeWarnings: string[]
}
```

```ts type-equiv
/**
 * Durable Team state published to browser clients through the Lead Session's
 * `agentTeam` projection. `failure` names the first rejected persisted Team
 * record; members and tasks then stay at the last valid state.
 */
interface TeamProjection {
  readonly members: TeamMemberProjection[]
  readonly tasks: TeamTaskView[]
  readonly failure?: string
}
```

## リプレイ

`agentTeam` Session projection は 1 つのルート Session をロスター、タスクボード、歴史的なキュー済みマイナス配信済みメールボックスにリプレイする。`TeamId` でレコードを選択するため、通常のフォークによって継承されたイベントは祖先 id を保持し、新しいルートの状態に決して入らない。Session イベントの `seq` と `time` は順序とタイミングのレコードのままであり、Team スナップショットはそれらを複製しない。ロスターとタスクの読み取りはビューとして呼び出し元に届く。保留メールは呼び出し元に公開されない。パッケージ [README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/agent-team/README.md) は操作、認可、復旧、制限の動作を所有する。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxagentteams--teamservice"></a>

### `ctx.agentTeams` — `TeamService`

正確なライブ Lead Session ログに裏付けられた Agent Teams サービス。

```ts cordis-catalog
/**
 * Resolve one exact live Agent's Team role.
 * @param agent - exact live Agent used as the authority credential.
 * @returns its root, Team identity, role, and model-facing name.
 */
membership(agent: Agent): TeamMembership

/**
 * List the runtime-enriched roster visible to one Team member.
 * @param agent - exact live Team member.
 * @returns Lead and teammate rows in creation order.
 */
listMembers(agent: Agent): TeamMemberView[]

/**
 * Create one named, continuable direct child of the Team Lead.
 * @param caller - exact live Lead Agent.
 * @param request - immutable name, description, prompt, context mode, provider, and cancellation.
 * @returns the active roster row.
 */
async spawnTeammate(caller: Agent, request: SpawnTeammateRequest): Promise<SpawnTeammateResult>

/**
 * Steer one peer message into the target inbox or reject the attempt.
 * @param caller - exact live sending Team member.
 * @param request - target name, content, and cancellation before acceptance.
 * @returns accepted inbox identity; acceptance follows normal Agent persistence and does not await model processing.
 */
async sendMessage(caller: Agent, request: SendTeamMessageRequest): Promise<SendTeamMessageResult>

/**
 * Create one unowned pending task in the Team Lead log.
 * @param caller - exact live Team member creating the task.
 * @param request - task text, blockers, and advisory write scopes.
 * @returns the revision-one task view.
 */
async createTask(caller: Agent, request: CreateTeamTaskRequest): Promise<TeamTaskView>

/**
 * Return one task, including a deleted tombstone.
 * @param caller - exact live Team member reading the task.
 * @param id - Team-local task identity.
 * @returns the latest task value and derived readiness diagnostics.
 */
getTask(caller: Agent, id: TeamTaskId): TeamTaskView

/**
 * List current non-deleted tasks in numeric creation order.
 * @param caller - exact live Team member reading the board.
 * @returns detached current task views.
 */
listTasks(caller: Agent): TeamTaskView[]

/**
 * Compare-and-set one authorized task transition.
 * @param caller - exact live Team member authorizing the mutation.
 * @param request - task identity, expected revision, action, and action fields.
 * @returns the committed next task revision.
 */
async updateTask(caller: Agent, request: UpdateTeamTaskRequest): Promise<TeamTaskView>

/**
 * Wait for the next Team-domain or member-status change.
 * @param caller - exact live Team member waiting for activity.
 * @param timeoutMs - bounded wait duration from ten seconds through one hour.
 * @param signal - caller cancellation for the wait only.
 * @returns one observed change or a timeout result.
 */
async waitForChange(caller: Agent, timeoutMs: number, signal: AbortSignal): Promise<TeamWaitResult>

/**
 * Interrupt one live teammate turn without clearing its pending inbox.
 * @param caller - exact live Lead Agent.
 * @param targetName - durable teammate name.
 * @returns the target status sampled before cancellation.
 */
interrupt(caller: Agent, targetName: string): { previousStatus: 'running' | 'inactive' }

/**
 * Resolve a caller without throwing, used by scoped-tool installation and observers.
 * @param agent - candidate exact live Agent.
 * @returns Team membership, or undefined for non-Team subagents and stale identities.
 */
tryMembership(agent: Agent): TeamMembership | undefined
```

型: [Agent](core.md)

ソース: [`packages/experimental/agent-team/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/agent-team/src/index.ts)
<!-- END GENERATED cordis-surface -->
