# Conversation の組み立て

日本語 | [English](../../../docs/subsystems/conversation.md) | [中文](../../../docs/subsystems/conversation.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/conversation.md](../../../docs/subsystems/conversation.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

Conversation は、Client の `SessionEventLikeEntry` ウィンドウとブラウザ・ビューとの間にあるターゲット中立の組み立て層である。[`ui-conversation`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-conversation/README.md) はイベント・レジストリとビュー・レジストリ、`SessionBinding` ごとの同一性安定なバインディング、Turn/Step の位置、増分 Context 組み立て、ターゲット・ソース、共有シェル、入力オーケストレーションを所有する。[`ui-chat`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-chat/README.md) や [`ui-trajectory`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-trajectory/README.md) などのターゲット・パッケージは、自身の Definition、最終スナップショット、描画を所有する。

このページは、ビジネス所有の Conversation ノードのデータ・モデルと拡張パスを定義する。[Web Client アーキテクチャ](web-client.md) はこのサブシステムを Client モデルと Slot の間に位置づけ、[Conversation Node 組み立ての決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-09-client-conversation-node-assembly.md) がその根拠を所有する。

## データ・モデルと所有権

Session Controller は、連続してロードされた論理イベント・ウィンドウを所有する。各 `SessionEventLikeEntry` は、1 つの永続イベントに対する `{ type: 'event', event: SessionEvent }` か、1 つの Client 専用 `assistant/live-chunk` プレゼンテーションに対する `{ type: 'transient', event: AssistantLiveChunkEvent }` のいずれかである。両方の内側イベントは `type`、`seq`、`time`、`data` を公開する。`ui-conversation` は 2 つ目の履歴ストリームを開かずに、これらのエントリをアセンブラーへ渡す。Session ごとに 1 つの `ConversationNodeAssembler` が、登録済みのすべての Definition を適用し、登録済みの各ビュー・ターゲットに対して独立したソースを公開する。

| 概念 | 所有者と目的 |
|---|---|
| イベント Definition | ビジネス・パッケージが、一度に 1 つの永続イベントまたは Client 専用一時イベントに一致し、安定した `(kind, id)` で相関させ、決定的な State を畳み込み、任意で 1 つのターゲット・ノードを具現化する。 |
| Context | 1 つの `(kind, id)` に対する、エンジン所有の順序付き Match と現在の State。永続および一時イベントが開始になり得る。最も早いロード済み開始が State を初期化し、以降の Match がそれを更新する。更新のみの証拠は、その開始がロードされるまで保留のままである。 |
| Location | 永続バウンダリ・イベントから導出される、エンジン所有の Session、Turn、Step の座標。Definition は型付きデータを 1 つの Turn または Step に公開してよい。 |
| View Definition | ターゲット・パッケージが Session ごとに 1 つの増分ビルダーを作成し、そのターゲットの最終スナップショット型を所有する。 |
| Group Definition | ビジネス・パッケージが、具現化された Node から 1 つのターゲットのルート参照とグループ・スナップショットを導出し、メンバーシップ、セグメント化、サマリー、増分キャッシュを所有する。 |
| View | Chat や Trajectory のような Slot エントリが、自身のターゲット・スナップショットだけを読み、ターゲット所有の Node を描画する。 |

Chat と Trajectory は同じ永続イベント・ファミリーを認識してよいが、それぞれが独自の Definition State と最終ノード・ペイロードを保つ。共有されるターゲット中立の機構は、同一性ルーティング、順序付きリプレイ、Location データ、先行依存、公開ケイデンスに限られる。

## ターゲットのアクティブ化

各 Session は単調なアクティブ・ターゲット集合を保つ。ターゲット・ソースの作成や読み取りはそれをアクティブ化しない。シェルは永続化された、または新しく選択された View を明示的にアクティブ化し、別のコンシューマーは最初のソース・サブスクリプションを通じてターゲットをアクティブ化する。最初のアクティブ化はそのターゲットのビルダーを作成し、現在のターゲット索引付き Context から `replace()` を 1 回呼ぶ。以降のフラッシュはすべてのアクティブ・ターゲットに対して `apply()` を呼び、サブスクリプション解除はアクティブ化を解除しない。

シェルは View の選択を所有し、バインディングが作成または現在として選択されたとき、および View 名簿の変更後に、描画前に登録済みの優先 View または Chat フォールバックを解決する。アセンブラーは解決済みのターゲット id だけを受け取り、Chat や別のデフォルト・ターゲットを選択しない。サードパーティの View は同じ選択とアクティブ化の操作を通じて参加する。View Definition は `toolCallFocus(callId)` を公開してよい。シェルはこのケイパビリティを宣言する可視ターゲットにのみ Inspect を提供し、ターゲットは call id を自身のフォーカス同一性にマップする。

<a id="group-definitions"></a>
## グループ定義

任意の `ctx.uiConversation.groups.register(definition)` コントリビューションは、イベントおよびビューの登録と同じエフェクト所有のレジストリ・ライフサイクルを使う。既存の View ターゲットを必要とし、重複するターゲット登録は Builder を構築せずに拒否する。各 Session はそのターゲットがアクティブ化したときにのみ、独自の Group コンテキストを作成する。登録の置換は古いコンテキストを破棄し、既存のレジストリ再構築の間にその観測済みグループをクリアする。

[グループ型](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-conversation/src/client/contract/groups.ts) が完全なプロトコルを定義する:

| 型 | 意味 |
|---|---|
| `ConversationGroupDefinition<Node, State, Data>` | `create()` が Session ローカルの State を初期化する。`update(context, input)` が次の State を返す。`buildGroups(context)` が保留中の出力または `null` を返す。置換入力はエントリと完全なグループ置換を要求する。 |
| `ConversationGroupInput<Node>` | `replace` はターゲット順序、タイムライン、同期の `readNode`、`readTurn`、`readPosition` リーダーを供給する。`apply` は投影された `previous/current` Node 変更、ライフサイクルの `changedTurns`、`changedTurnOrders` を追加する。リーダーを State に保持してはならない。 |
| `GroupNodePosition` | 所有 Turn（存在する場合）と、直前・直後の可視 Node キー。隣接は、Turn のみのキー・リストが省略する中断を保持する。 |
| `NodeReference` / `GroupReference` | `kind` で区別されるブランド付き `NodeKey` または `GroupKey`。Node 参照はレンダラー所有の `groupPart` を選択してよく、省略は Node 全体を選択する。 |
| `GroupSnapshot<Data>` | 不変のグループ・キー、ビジネス・データ、順序付き Node 参照。グループは他のグループを含めず、ソース Node データを所有しない。 |
| `GroupUpdate<Data>` | `entries` はルート・シーケンス全体を置換し、apply 入力だけがシーケンスを保つためにそれを省略できる。置換入力は `entries` と `groups.replace` の両方を要求する。`groups.replace.snapshots` はすべてのグループを置換し、`groups.apply.upserts/removes` は名指されたグループ・レコードだけを更新する。削除は Node を削除しない。 |
| `ConversationGroupDataMap` | 登録と `views.grouped(target)` が共有する、宣言マージされたターゲット→データの対応。宣言されていないターゲットはグループ・ペイロード型を持たない。 |
| `ConversationGroupedView<Data>` | 安定したルート `entries` とキー付き `groupSource(key)` リーダー。削除されたグループは `undefined` として読まれる。 |

Builder はグループ化が登録されている場合に `groupInput()` を公開し、入力の欠落は最初のアクティブ化で失敗する。自身の projection の後の Node 値を記録し、コンテンツのみの順序同一性を保つ。アセンブラーはすべての更新で変更された Turn を供給し、グループ化されていない直接の Builder 呼び出し元はそれらを省略してよい。独立したソースを持つ Builder は、通知を `publish()` へ延期する。最初のアクティブ化と通常のフラッシュは同じシーケンスを共有する。Location データと Node を具現化し、Builder を更新し、Group Definition を呼び、グループ化を検証してインストールし、影響を受けるすべてのターゲットと Location ソースを公開する。グループ・フェーズは既存の公開ケイデンスを再利用する。

ターゲット位置インデックスはリーダーを供給し、可視キーまたは隣接が変わった Turn を識別する。`changedTurnOrders` は、移動した Node の両方の所有者と、挿入または削除されたスコープ外 Node に隣接する Turn を含む。ライフサイクルのみの変更は `changedTurns` に残る。ビジネス Definition はそれらの Turn を再セグメント化し、変更されたコンテンツを含むグループだけを更新できる。構造的出力は依然としてルート参照配列全体を置換するが、変更されていない Node コンテンツを再読する必要はない。

Group ストアはインストール前に提出された結果全体を検証する。ルート Group 参照とレコードは 1 対 1 で対応し、参照されたすべての Node が存在し、各 `(NodeKey, groupPart)` は最大 1 つのルートまたはメンバー位置を占める。Node 全体はそのパーツの 1 つと共存できない。重複する upsert、重複する削除、グループの同時 upsert/削除は失敗する。データのみの upsert はルートとメンバー配列を保ち、Node を読まず、変更されたグループ・ソースだけに通知する。完全置換はすべての参照を再検証する。

レンダラーは 2 つの参照種別で分岐し、グループ・データ内のレンダラー・フィールドではなく、所有 View 内でコンポーネントを選ぶ。ルート Node 参照と参照されたグループ・メンバーが完全な描画リストを形成する。参照されていないターゲット Node は保存されたまま描画されない。グループ本体はサマリー・データとは別にメンバーを読む。プレゼンテーション・モードはコンポーネント型、キー、メンバー親を保持しなければならない。データのメンバーシップ変更は、移動したメンバーを正当に再マウントしてよい。フレームワークはパーツの完全性、セグメント化、プレゼンテーション・ポリシーを解釈しない。

## リプレイ可能なイベント・ファミリー

Definition を書く前に、安定したビジネス id を 1 つ選ぶ。同じ Node に寄与するすべてのイベントは、その id を運ぶか、自身のペイロードから独立に導出しなければならない。クライアントは更新を「最新の未完了」Context に割り当てては決してならない。

レビュー・ジョブの場合、イベント契約は次のようになる:

| イベント | 役割 | 必須の永続ファクト |
|---|---|---|
| `review/start` | 一意な開始 | `reviewId`、Turn/Step 座標、タイトル |
| `review/progress` | 更新 | 同じ `reviewId`、座標、リプレイ可能な進捗 |
| `review/end` | 更新 | 同じ `reviewId`、座標、最終サマリー |

プロセス境界を越えてプロデューサー所有のブランド付き id 型を使う。`SessionEventMap` のマージとペイロード型をプロデューサーの型専用エクスポートに置き、クライアント・パッケージからそのエクスポートを副作用のためにインポートする。各 `(kind, id)` は最大 1 つの開始イベントを持てる。単一イベントのビジネスは、`event.seq` のようなイベントの安定した同一性を Definition ローカル id として使える。

増分イベントはサポートされる。プロデューサーが安価に発行できるなら、値全体のチェックポイントを優先する。開始がロード済みウィンドウの外にある場合でも有用だからである。各デルタは安定した id を運び、ログの `seq` 昇順でリプレイしたときに決定的な State を生成しなければならず、ライブ専用のメモリに依存してはならない。現在の履歴ウィンドウが更新だけを含む場合、アセンブラーは保留中の Context を保ち、古いページが開始を供給するまで State を構築しない。開始がロードされる前に描画しなければならない場合、終端またはチェックポイント・イベントが、Definition がその結果を直接構築できるだけの全量フォールバック状態を運ばなければならない。無関係なイベントを走査して回復してはならない。

ライブの Assistant デルタは Client 専用の `assistant/live-chunk` イベントとして届く。再接続ベースラインは、アクティブなプロセス・ローカルのコンパクト・ストリームを同じ一時イベントへ展開する。一方、永続的な `assistant/message` と `assistant/attempt` イベントは履歴リプレイ用の完全なコンパクト・ストリームを埋め込む。一時イベントは Context を初期化してよい。すべてのツール・デルタとその後の `tool/call` は、名前が届く前でも同じ callId の下で開始として一致できる。最も早いものだけが `start()` を呼び、後の Match は `update()` を呼ぶ。準備中の Tool ノードは、名前が利用可能になった後でのみ可視になる。履歴ページは決着済みメッセージをライブ・デルタに展開しない。Assistant 出力を消費する Definition は、ライブ・チャンクと永続的な決着を同じマッチングと更新のインターフェースを通じて処理し、無関係な Definition はストリームを展開せずに `null` を返す。

## 定義と型付き Chat ペイロード

この例は、プロデューサー宣言とクライアント・コントリビューションを 1 つのブロックに置き、関係全体が見えるようにしている。パッケージ・ファミリーでは、ブランド付き id と `SessionEventMap` 宣言をイベント・プロデューサーに置き、Definition、Chat データ・マージ、レンダラーをクライアント・プラグインに置く。

```ts ignore-check
import { createElement } from 'react'
import type { Context as ClientContext } from '@deepseek-ai/cordis'
import type { Branded } from '@deepseek-ai/dsh-brand'
import type {
  ConversationLocation, ConversationNodeContext,
  ConversationNodeDefinition,
} from '@deepseek-ai/dsh-client-ui-conversation/client'
import type { ChatNodeViewProps } from '@deepseek-ai/dsh-client-ui-chat/client'

type ReviewId = Branded<'ReviewId'>

interface ReviewStartData {
  readonly reviewId: ReviewId
  readonly turn: number
  readonly step: number
  readonly title: string
}

interface ReviewProgressData {
  readonly reviewId: ReviewId
  readonly turn: number
  readonly step: number
  readonly completed: number
}

interface ReviewEndData {
  readonly reviewId: ReviewId
  readonly turn: number
  readonly step: number
  readonly summary: string
}

declare module '@deepseek-ai/dsh-session/types' {
  interface SessionEventMap {
    /**
     * Opens one durable review job.
     * @mode emit
     * @param data - stable identity, location, and initial display state.
     */
    'review/start': ReviewStartData
    /**
     * Records replayable progress for one review job.
     * @mode emit
     * @param data - stable identity, location, and latest progress.
     */
    'review/progress': ReviewProgressData
    /**
     * Closes one review job with its final summary.
     * @mode emit
     * @param data - stable identity, location, and final display state.
     */
    'review/end': ReviewEndData
  }
}

interface ReviewChatData {
  readonly title: string
  readonly completed: number
  readonly status: 'running' | 'completed'
  readonly summary?: string
}

declare module '@deepseek-ai/dsh-client-ui-chat/client' {
  interface ChatNodeDataMap {
    'review-job': ReviewChatData
  }
}

declare module '@deepseek-ai/dsh-client-ui-conversation/client' {
  interface ConversationStepDataMap {
    'review-job': ReviewChatData
  }
}

interface ReviewState extends ReviewChatData {
  readonly turn: number
  readonly step: number
}

function locationOf(context: ConversationNodeContext): ConversationLocation {
  return context.start?.location ?? context.matches[0]?.location ?? { kind: 'unresolved' }
}

function viewData(state: ReviewState): ReviewChatData {
  return {
    title: state.title,
    completed: state.completed,
    status: state.status,
    ...state.summary === undefined ? {} : { summary: state.summary },
  }
}

const reviewDefinition: ConversationNodeDefinition<ReviewState> = {
  kind: 'review-job',
  target: 'chat',
  match: (event) => {
    if (event.type === 'review/start') {
      return { id: String(event.data.reviewId), role: 'start' }
    }
    if (event.type === 'review/progress' || event.type === 'review/end') {
      return { id: String(event.data.reviewId), role: 'update' }
    }
    return null
  },
  start: (_context, match) => {
    if (match.event.type !== 'review/start') throw new Error('review-job requires review/start')
    return {
      turn: match.event.data.turn,
      step: match.event.data.step,
      title: match.event.data.title,
      completed: 0,
      status: 'running',
    }
  },
  update: (context, match) => {
    if (match.event.type === 'review/progress') {
      return { ...context.state, completed: match.event.data.completed }
    }
    if (match.event.type === 'review/end') {
      return { ...context.state, completed: 100, status: 'completed', summary: match.event.data.summary }
    }
    return context.state
  },
  publication: match => match.event.type === 'review/progress'
    ? 'animation-frame'
    : 'immediate',
  buildLocationData: (context, scope) => {
    if (scope !== 'step' || context.state === undefined) return null
    return {
      kind: 'step',
      turn: context.state.turn,
      step: context.state.step,
      key: 'review-job',
      value: viewData(context.state),
    }
  },
  buildViewNode: (context) => {
    if (context.state === undefined) return null
    return {
      key: context.key,
      kind: 'review-job',
      id: context.id,
      target: 'chat',
      anchorSeq: context.start?.event.seq ?? context.matches[0]?.event.seq ?? 0,
      location: locationOf(context),
      visibility: 'visible',
      data: viewData(context.state),
    }
  },
}

function ReviewNodeView({ node }: ChatNodeViewProps<'review-job'>) {
  const text = node.data.summary ?? `${node.data.title}: ${node.data.completed}%`
  return createElement('p', null, text)
}

export const inject = ['uiConversation', 'slots']

export function apply(ctx: ClientContext): void {
  ctx.uiConversation.events.register(reviewDefinition)
  ctx.slots.inject('conversation.chat.node', () => ctx.slots.register({
    name: 'conversation.chat.node',
    key: 'review-job',
  }, ReviewNodeView))
}
```

`match(event)` はフォールドではなく同一性抽出器である。現在の `SessionEventLike` だけを受け取り、Definition ローカル id とライフサイクル役割を返す。マッチ後、アセンブラーは `(kind, id)` で Context を特定し、永続か一時かを問わず、最も早いロード済み開始から初期化する。以降のすべての Match は、別の開始であっても `update` を呼ぶ。一時的な Match を削除すると、残りの証拠から開始を再選択して State を再計算する。残る開始がなければ State は undefined のままである。両方の関数はエンジンが採用する State を返す。新しい不変値を返すのが望ましいが、同じオブジェクトを変更して返す関数も同じ採用セマンティクスを持つ。

`ConversationNodeDefinitionInput` は、この関数か、イベント型からマッチング関数への読み取り専用テーブルのいずれかを受け付ける。関数形式の登録はすべてのイベントを受け取り、テーブル登録は列挙された型だけを受け取る。レジストリは登録が変わったときにテーブル・エントリをコピーし、順序付き候補 Set を事前計算して、候補構築をイベントごとのディスパッチの外に保つ。テーブルを置換するには、登録解除してから再登録する。解決済みの `ConversationNodeDefinition` 値は、どちらの場合も呼び出し可能な `match(event)` を保つ。

`exclusive: true` を持つマッチ結果は、Definition のターゲット上でそのイベントを要求する。アセンブラーはイベントのすべてのマッチを評価してからいずれかを受け入れるため、結果は登録順に依存しない。要求されたターゲットの他の Definition はそのイベントの Context を構築せず、ターゲットの未マッチ・イベント・フォールバックも実行されない。他のターゲットの Definition とターゲットを持たない Definition は依然としてそれを受け取る。1 つのターゲットに対する 2 つの要求、またはターゲットを持たない Definition からの要求はディスパッチ中にスローする。フォールバック自身が返す要求は効果を持たない。ビジネス Definition は、同じターゲットの汎用 Definition がそうしなければ提示してしまう種類のメッセージ（たとえば遅延した質問への返信）を提示するために要求を使う。

`buildLocationData(context, scope)` は任意で、Definition 所有のデータをエンジン所有の Turn または Step に公開する。宣言マージを使って各キーに正確な値型を与える。同じ Location 内の別の Node は、`useTurnData(key)` のような制約付きスロット・フックを通じてその値を消費でき、Session を受け取ったり `snapshot.chat.nodes` を走査したりする必要はない。

`target` と `buildViewNode(context)` は 1 つのターゲット所有の描画コントリビューションを宣言し、一緒に現れなければならない。`context.key` を React に面する同一性として保ち、マッチしたイベントの順序から `anchorSeq` を選び、レンダラー準備完了のデータだけを返す。ターゲット Node が公開された後は、同じキーを返し続ける。可視フローから一時的に外す必要がある場合は `null` で撤回するのではなく、`visibility: 'hidden'` を使う。

Chat Node は `groupTree` を通じて、その行がどこに立つかも選ぶ。位置は最も外側から内側へ順序付けられ、ある位置の Node はすべての内側の位置の効果も持つ。Chat のプロセス・グループ化、Turn 全体の折り畳み、Turn 入力の検出はこの位置だけを読む。[group-tree.ts](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-chat/src/client/contract/group-tree.ts) が完全な契約を保持する。

| Node の性質 | 宣言 |
|---|---|
| Step 1 で要求された人の入力。その Turn が交換を開く | `exchange` |
| その Turn の別の入力（後からの人間入力やトリガー通知など）。プロセスの前では開始入力であり、その後は Turn 全体が開いたままになる | `input` |
| Turn 全体の折り畳みの外に留まる Turn 通知（エラーや完了 turn のフッターなど） | `turn` |
| Turn 全体の折り畳みが依然として隠す独立した行（モデルの再試行など） | `process` |
| 周囲の step 実行に参加する通常のプロセス・コンテンツ | `groupTree` を省略 |

[Chat ビジネス・ルール](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-chat/src/client/conversation-nodes/README.md#process-grouping) は組み込み Node が宣言する位置を列挙する。同じターゲットの汎用 Definition が、あなたの Node が提示するイベントにも一致する場合、上記のように `exclusive: true` でそのイベントを要求する。

## 先行読み取り

一部の Definition は、別のビジネス種別の最新の以前の State を必要とする。`start` は `ConversationContextReader` を受け取る。Context コレクションを受け取ったりイベントを走査したりする代わりに、そこで `reader.previous<State>(kind)` を呼ぶ。リーダーは現在の開始 `seq` より前の最も近い開始済み Context を読み取り専用データとして返す。

アセンブラーはその依存を記録する。古いプリペンドが後に、より近い先行を供給するか、以前は不明だったウィンドウ・ギャップを閉じるか、先行 State を改訂すると、依存する Context を `start` から再実行し、その更新を `seq` 昇順でリプレイする。問い合わせられた Definition は有用な State を書く責任を負ったままである。リーダーはビジネス固有のクエリ・メソッドを公開せず、別の Context に対する変更権限も与えない。

## ウィンドウ更新パス

履歴は末尾から 1 ページずつ遡って要求できる。Session ジャーナルはまず重複しない論理シーケンス範囲を検証し、その後アセンブラーが受け入れ済み入力を最初の `seq` で順序付けてから State をリプレイする。

| パス | エンジンの作業 | Definition から見える挙動 |
|---|---|---|
| 開く、再同期、ギャップ修復時の置換 | ロード済みウィンドウを再構築し、すべての標準イベントまたはパック実行を Definition ごとに 1 回マッチさせ、開始済みの各 Context をリプレイする | `start` に続き、論理 `seq` 昇順でその更新。更新のみの保留 Context は State なしのまま |
| 古いページを 1 つプリペンド | 新しい古い入力だけをマッチさせ、`(kind, id)` で Context にマージし、既存のキー付きノードを保ち、影響を受ける Context と依存だけをリプレイする | 新たに見つかったスカラー開始が、収集済みのスカラー更新とパック更新を有効化する。変更された Location または先行が Context を再実行することがある |
| ライブ・イベントを 1 つアペンド | 各 Definition の `match` を 1 回呼び、マッチした Context をキーで検索し、その Context だけを更新する | 開始後のイベントに一致する場合、1 回のスカラー `update` と 1 回の要求された公開。既存 Context の走査はない |

`D` 個の登録済み Definition がある場合、1 つの受信スカラー・イベントまたはパック実行は、`D` 回の現在入力マッチと、マッチ後の定数時間 Context キー・ルックアップを行う。Definition コードはその性質を保たなければならない。通常の追記パスで、イベント・ウィンドウ全体、すべての Context、`context.matches`、描画済み Node コレクションを走査してはならない。蓄積されたファクトには State、同じ Turn/Step 間の共有には Location データ、索引付き先行依存には `reader.previous()` を使う。

`publication` は変更された State がいつ具現化されるかを制御する。構造的または終端的な変更には `immediate`、高頻度の可視デルタには `animation-frame`、State 変更が後の公開にのみ供給される場合は `none` を使う。エンジンはすべてのスカラー更新をログ順に適用し、すべてのパック実行を 1 つのバッチ更新として適用する。ケイデンスはビューの公開を合流させるだけである。

## 検証の義務

次の結果を確立する焦点を絞ったテストを追加する:

1. replace を通じて渡された完全なウィンドウが、期待される最終 State、Location データ、Node ペイロード、`anchorSeq` を生成する。
2. 更新のみの末尾は保留のままで、その開始をプリペンドすると完全な replace と同じ結果を生成する。
3. 初期履歴の後のライブ・アペンドは、結合されたウィンドウのリプレイと同じ結果を生成する。
4. 古いページのプリペンドは、データが変わっていない既存のキー付き Node 値を置換することなく、より早い行を追加する。
5. 可視デルタの繰り返しは `context.key` を保ち、要求された場合はアニメーション・フレームごとに最大 1 回公開する。
6. キー付きレンダラーは `node.data` と制約付き Location フックだけを消費し、Session イベント・ウィンドウ、Context、Chat Node を走査しない。
7. スカラーおよびパックの Assistant 履歴は、同じ最終 State、タイミング境界、ターゲット・スナップショットを生成する。一方、1 つのパック実行は replace、プリペンド、Location リプレイ、レジストリ再構築を通じて 1 つの Match のままである。
8. ターゲット・ソースの作成はビルダー作業を行わない。明示的な選択または最初のサブスクリプションが 1 回の完全な置換を行い、以降の更新はすべてのアクティブ・ターゲットに届き、繰り返しのアクティブ化は置換を行わない。

ストリーミングと中断には [`packages/client/ui-chat/src/client/conversation-nodes/assistant.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-chat/src/client/conversation-nodes/assistant.ts) を、先行クエリには [`inbox.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-chat/src/client/conversation-nodes/inbox.ts) と [`message.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-chat/src/client/conversation-nodes/message.ts) を、自身の Node を作成せずに Turn データを公開する Definition には [`packages/client/ui-deliverables`](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/client/ui-deliverables) を使う。
