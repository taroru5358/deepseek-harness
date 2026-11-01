# 会話と Session のルール

日本語 | [English](../../../docs/defensive-patterns/conversation-and-session.md) | [中文](../../../docs/defensive-patterns/conversation-and-session.zh.md)

> これは非公式の日本語訳です。英語版（[docs/defensive-patterns/conversation-and-session.md](../../../docs/defensive-patterns/conversation-and-session.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

Session の読み取り、保持、会話ノードの計算、およびそのレンダリングに関するルールです。データ・モデルと型ごとのセマンティクスは [session-projection.md](../subsystems/session-projection.md) と [conversation.md](../subsystems/conversation.md) にあります。

## 読み取りと計算の経路は Projection だけである

Session のデータは Projection を通じて読み取り、計算します。Client は Session ログを引き出すのではなくプロジェクション値を読み、Session に属する計算は、要求に応じて再計算されるのではなく、結果を発行する登録済みのプロジェクション・ユニットとして実行されます。同じ問いへの 2 つの答え（1 つは pull、1 つは push）は乖離するため、レンダーが、Session ログがもはや裏付けない値を表示することがあります。

- 判定条件: Client がレンダリングするすべての Session 由来の値は、レンダー位置でのプロジェクション読み取りを通じて届き、Session 内の新しい計算はすべてユニットとして登録され、その結果が発行された値として Client に届くこと。Session データを自身でフェッチするコンポーネント、または読み取り側が要求時に起動する計算が違反です。
- 状態とビュー: ユニットはホスト側の State を畳み込み、その上にクライアント可視の View を宣言し、登録がその分割を明示します。`ProjectionDefinition` のフィールドとそれを駆動するレジストリは [session-projection.md](../subsystems/session-projection.md#ctxsessionprojections--sessionprojectionregistry) に属します。
- バージョン: `stateVersion` は 1 から始まり、シリアライズされた状態フィールドまたは畳み込みのセマンティクスが変わるたびに、互換であってもなくても上げます。互換な変更が可能な場合はそれを優先しますが、それでも上げます。1 つのキーを異なる `stateVersion` 値で 2 回登録すると、マージせずに throw します。
- 計算戦略: ユニットは既定で全体を計算します。ユニットが計算する範囲を狭めるには所有ドメインの専門家の確認が必要で、実装者自身の判断では決してありません。
- 境界: 部分的なプロジェクション・ビューは、接続された Session が応答する前に描画されるキャッシュ済みブロックのように、個々のキーのインターフェース可用性に使えます。外部から見える結果を決めることはなく、それは標準のプロジェクション値を待ちます。

自身のレンダー位置で読むコンポーネントと、引き出されるのではなく登録する計算は、どちらも準拠します:

```tsx
// The client reads the published value; it never fetches Session data itself.
function PlanChip({ useProjection }: PlanChipProps) {
  const plan = useProjection('plan')
  return plan === undefined ? null : <Chip label={plan.label} />
}

// The computation is a registered unit; the registry drives it and publishes changes.
const planDefinition: ProjectionDefinition<'plan', typeof emptyPlan> = {
  key: 'plan',
  stateVersion: 1,
  init: () => emptyPlan,
  apply: (state, event) => foldPlan(state, event),
  wire: { viewSchema: planSchema, view: planView },
}
```

同じ値を独自のリクエストで引き出すコンポーネントは、単一の読み取り経路を壊します:

```tsx
function PlanChip() {
  const [plan, setPlan] = useState<Plan>()
  useEffect(() => {
    void fetchSessionPlan(sessionId).then(setPlan)   // a second read path beside Projection
  }, [sessionId])
  return plan === undefined ? null : <Chip label={plan.label} />
}
```

引き出されたコピーと push された値が食い違い、どのバージョン・チェックも Session が現在どちらを保持しているかを判別できません。

## コールド・リードは最後の手段である

利用可能なキャッシュを元のログより先に読み、プロジェクションがデータをカバーしていない場合にだけコールド・リードに手を伸ばします。「小さな Session は全体を読んでよい」という例外は削除済みで、再検討されません。キャッシュされたデータと公式のデータが等価でない場合、行の欠落や古さは明示的な結果を生み、黙って遅い経路へ切り替わることはありません。

- 判定条件: Session の読み取りがプロジェクション・キャッシュを最初に参照し、行が存在しない場合は呼び出し元に明示的な状態を残すこと。`else` 分岐が完全なログを開くキャッシュ・テストが違反です。
- フォールバックのセマンティクス: 呼び出し元がミスの意味を決め、戻り値で明示します。キャッシュ・サイズや Session サイズの例外はなく、その承認者もいません。
- 境界: この制限は Client のものです。ホストに対する絶対的な「コールド・リードを分岐させるな」は要求されません。ホストは Client が開けないログを開く必要があるかもしれないからです。

保存されたプロジェクション行を最初に読み、ミスは呼び出し元がレンダリングする状態のままになります:

```ts ignore-check
// The listing reads stored projection rows; a miss remains an explicit state.
type ListingView =
  | { kind: 'cached'; snapshot: ProjectionSnapshot }
  | { kind: 'unknown' }

function readListing(header: SessionHeader, keys: readonly ProjectionKey[]): ListingView {
  const cached = ctx.sessionProjectionCache.cachedSnapshot(header, keys)
  return cached === undefined ? { kind: 'unknown' } : { kind: 'cached', snapshot: cached }
}
// Opening the complete log is a separate operation the caller names, never this fallback.
```

完全なログへフォールスルーするキャッシュ・テストは、遅い経路を通常の経路に変えます:

```ts ignore-check
function readListing(header: SessionHeader, session: Session): ListingView {
  const cached = ctx.sessionProjectionCache.cachedSnapshot(header)
  if (cached !== undefined) return { kind: 'cached', snapshot: cached }
  return coldReadCompleteLog(session)   // every miss pays the full-log read
}
```

すべてのミスがフォアグラウンドでログ全体を読むようになるため、遅い Session はそれを要求する経路だけでなく、すべての一覧で遅くなります。

## 保持は 1 つのライフサイクル・ルールに従う

Client は明示的な参照（`SessionReference`）を通じて Session を保持し、参照カウントが 0 になると解放します。アイドル回収は 1 つのライフサイクル・ルール（参照がなく、処理中のものもない）に従い、ビジネス状態からどの Session を回収してよいかを列挙することは決してありません。そのリストは追加するすべての機能が保守しなければならないからです。暗黙の保持者は見えないため、回収側は誰がまだ Session を使っているかを判断できません。

- 判定条件: 述語は `refCount == 0 && inflight == 0` です。コードは acquire → use のウィンドウ全体で、retain と release を監査できる参照を通じて Session を保持しているか。モジュール・レベルの Session オブジェクトやグローバルな現在 id が違反です。
- 境界: 強制される部分は統一された述語とビジネス状態の列挙がないことであり、解放のトリガー自体は手作業で確認するため、手作業の確認だけを却下理由にはできません。
- 所有: `retain`、`release`、`ready`、および teardown の順序は [Session Controller のクライアント参照](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/session-controller/README.md#client-references) に属します。メッセージ内で運ばれるソース Session の言及は別の型であり、[session-reference.md](../subsystems/session-reference.md) が所有します。

明示的な参照を通じて保持し、teardown で解放するビューは準拠します:

```ts ignore-check
class SidebarSessionView implements Disposable {
  private readonly reference: SessionReference

  constructor(readonly id: SessionId, sessions: ISessions) {
    this.reference = sessions.retain(id, { source: 'sidebarView' })
    void this.reference.ready.catch((error: unknown) => { console.error('opening failed:', error) })
  }

  [Symbol.dispose](): void {
    this.reference.release()   // the last reference releases the Session
  }
}
```

Session オブジェクトを保持し続けるモジュール状態は、すべての保持者を回収側から隠します:

```ts ignore-check
let currentSessionId: SessionId | undefined
const sessionCache = new Map<SessionId, Session>()

export function remember(session: Session): void {
  currentSessionId = session.id
  sessionCache.set(session.id, session)
}

export function reclaimIdle(): void {
  if (state === 'idle' || state === 'closed') reclaim()   // a newly added state is missed
}
```

誰がまだ Session を使っているかを数えられず、Session に追加された次の状態は列挙から漏れて回収されません。

## ノード計算は増分的で、ストリーミング発行はバッチ化される

新しいノード計算はその複雑さを明示し、高頻度のストリーミング・チャンクは即座に発行されず、全体を再計算することもありません。チャンクごとの全体再構築を一度に発行すると、ノード・コレクション全体がフレーム予算に載り、トランスクリプトがストリームの後ろで止まります。追加経路のコスト（`D` 個の登録済み Definition、`D` 回のマッチ、1 回の定数時間の Context キー・ルックアップ）と発行の語彙（`none`、`animation-frame`、`immediate`）は [conversation.md](../subsystems/conversation.md#definition-and-typed-chat-payload) が所有します。

- 判定条件: 1 つのチャンクが計算する量（イベント・ウィンドウではなく、影響を受けるサブツリーへの増分）と、その結果をいつ発行するか（チャンクごとに 1 回の発行ではなく、バッチ化ポリシー）。
- 境界: このルールは実装を拘束します。ケイデンスだけを却下理由にはできないため、変更はそれが明示するケイデンスと使用する発行経路で判定されます。
- 高頻度経路で禁止: `immediate` 発行と、チャンクごとの全体再計算。
- 境界: 構造的な変更や終端の変更は `immediate` を発行してかまいません。ケイデンスは発行ごとに選ばれるからです。バッチ化に拘束されるのは高頻度の差分経路だけです。

1 つの Context を更新し、合流された発行を要求するチャンクは準拠します:

```ts ignore-check
const assistantDefinition: ConversationNodeDefinition<AssistantState> = {
  kind: 'assistant-step',
  target: 'chat',
  match: event => matchAssistant(event),
  start: (_context, match) => initialState(match),
  update: (context, match) => appendChunk(context.state, match.event),   // increment, not the window
  publication: () => 'animation-frame',   // high-frequency deltas coalesce; never immediate
  buildViewNode: context => assistantNode(context),
}
```

チャンクごとにノード・コレクション全体を再構築し、即座に発行します:

```ts ignore-check
function onChunk(chunk: AssistantLiveChunk): void {
  const nodes = buildNodesFromCompleteWindow(session)   // full recompute per chunk
  target.publish(nodes, { publication: 'immediate' })   // and an immediate publish
}
```

各チャンクがフレーム予算の内側で全ウィンドウの再構築を要するため、グループ化とレンダリングがストリームの後ろで止まります。

## ノード構築とグループ化エンジンはビジネス・ルールを運ばない

ビジネス・ルールがグループ化エンジンに入ることはなく、ビジネス要件が構築済みペイロードを再形成することもありません。`ProcessState`、グループ成果物の階層、`conversation-nodes/process-groups.ts` にあるグループ・セマンティクスの集合はインフラストラクチャです。エンジンは grouped、standalone、reply のような列挙の配置値を理解し、どれが適用されるかは呼び出し元が計算します。エンジン内部の特殊ケースは、すべてのコンシューマーに対して生成される構造を変え、グループ計算の経路にビジネス条件を載せます。

- 判定条件: エンジンにビジネス条件が含まれるか。`ProcessState` フィールドの追加、グループ・レベルの追加や削除、1 つのビジネスのためのグループ・セマンティクスの変更を行う diff は不合格です。配置の決定は列挙値を計算する呼び出し元に属します。
- 境界: レイヤリング、そのコスト、既存のビジネス上の期待が、ビジネス側がグループ化の設計を作り直さない理由です。
- 境界: 新しい配置の意味を運ぶ新しい列挙値が、サポートされる拡張です。呼び出し元がどこで計算してもかまいません。
- 例外と承認者: 既定ではありません。このインフラストラクチャへの変更には対応するドメインの専門家の確認が必要です。[conversation.md](../subsystems/conversation.md#group-definitions) がグループ Definition の型を所有します。

呼び出し元がビジネスの意味を決め、エンジンが配置に使う列挙を渡します:

```ts ignore-check
// Business judgment stays in the caller.
function placementOf(command: CommandView): NodePlacement {
  return isHandoff(command) ? 'reply' : 'grouped'
}

// The engine consumes the enum as data; it reads no business condition.
function groupEntries(commands: readonly CommandView[]): readonly NodePlacement[] {
  return commands.map(placementOf)
}
```

エンジン自身の計算の内部にあるビジネス条件:

```ts ignore-check
// The grouping engine asks what a node means, not where it goes.
function placeProcessNode(node: ChatConversationViewNode): GroupPlacement {
  if (node.kind === 'command' && isRagMessage(node)) return 'rag'   // business special case
  return 'grouped'
}
```

生成される構造が 1 つのビジネスの意味を運ぶようになり、すべてのコンシューマーとグループ計算がその機能に従います。

## ノード・リストとシート・コンポーネントはビジネス分岐を運ばない

`ChatNodeList`、`ChatNodeSeat`、`ChatGroupSeat` はインフラストラクチャです。ビジネス条件もビジネス固有のフィールドも運ばず、1 つのビジネスがリストの走査やシートの選択を変えることはありません。新しいノード形状は、その kind を `ChatNodeDataMap` に提供し、kind の登録サーフェスを通じてレンダラーを提供するため、リストは kind でディスパッチし続け、ビジネスのセマンティクスはビジネス・モジュールに残ります。1 つの機能のために追加された分岐は、すべてのノードのレンダリングをその機能に依存させます。

- 判定条件: リストまたはシート・コンポーネント内のビジネス分岐、その props 上のビジネス専用フィールド、または 1 つのビジネスに奉仕する走査やシート選択の変更。
- 正しい出口: ビジネス・パッケージが自身のキーを `ChatNodeDataMap` にマージし、ノード kind のレンダラーを登録します。シートはノードの kind に対して登録されたコンポーネントをレンダリングします。登録の規律は [slots.md](../subsystems/slots.md#extension-rules) が所有します。
- 境界: すべてのビジネスに適用されるレンダリングの変更（順序付け、仮想化、アクセシビリティ）はビジネスの変更ではありません。
- 例外と承認者: 既定ではありません。例外は対応するドメインの専門家が確認します。

自身の kind を宣言し、レンダラーを登録するビジネス・パッケージは、共有リストを変更しません:

```tsx
// The business declares its kind and registers its renderer; the shared list is untouched.
declare module '@deepseek-ai/dsh-client-ui-chat/client' {
  interface ChatNodeDataMap { handoff: HandoffData }
}

const HandoffNodeView = memo(function HandoffNodeView({ node, t }: ChatNodeOwnerProps<'handoff'>) {
  return <HandoffCard data={node.data} t={t} />
})

ctx.slots.inject('conversation.chat.node', () => ctx.slots.register(
  { name: 'conversation.chat.node', key: 'handoff', locale: NS }, HandoffNodeView))
```

共有シートの内部にあるビジネス分岐とビジネス・フィールド:

```tsx
export const ChatNodeSeat = memo(function ChatNodeSeat({ node, handoffFlag }: ChatNodeSeatProps) {
  if (node.kind === 'user' && handoffFlag) return <HandoffSeat node={node} />   // business dispatch
  return <div data-handoff={handoffFlag}>{renderByKind(node)}</div>
})
```

1 つの機能の条件がすべてのノードのレンダリング方法を決めるようになり、次の機能のためには共有シートを再び変更しなければなりません。

## Markdown パイプラインはビジネス分岐を運ばない

`AssistantMarkdown` とそのレンダリング・パイプラインはインフラストラクチャです。ビジネス条件が Markdown のレンダー経路に入ることはなく、ビジネスがノード・マッピングやスタイル注入のルールを書き換えることもありません。プレゼンテーションの必要は既存のチェーンにレンダラーを登録するか、データ内で表現し、公式のデフォルト・コンテンツは、ホスト側のインライン・フォールバックを伴うオプションのレンダラーとして存在するのではなく、同じ登録サーフェスに参加します。ビジネスの特殊ケースをホストするパイプラインは、それ自体ではレビューも最適化もできなくなります。

- 判定条件: `AssistantMarkdown` またはそのブロック・マッピングの内部にあるビジネス分岐（たとえば `if (isXxxBusiness) renderXxx()`）、または 1 つのビジネスにキー付けされたスタイル注入ルール。
- 正しい出口: 既存のチェーンに登録されたレンダラー、または汎用レンダラーが読むデータ内のセマンティック・フィールド。Component Factory の登録は [ui-renderer の README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-renderer/README.md) が所有します。
- ラップ: 標準のレンダリングを必要とするラッパーは公式のエントリーを呼び、そのフォールバックを保ちます。Slot を再レンダリングすることも、パッケージをまたいでプライベートな React コンポーネントを import することもありません。
- 責任: ホストが位置と起動を所有し、プラグインが自身の状態、クリーンアップ、コントロールを所有し、公式ホストはコントリビューションのためのビジネス・アクションを運びません。
- `labels`: オプションの完全な `MarkdownLabels` オブジェクトはそのまま全体が使われ、既定値とのフィールド単位のマージはありません。省略すると公式の既定値を使います。`variant="compact"` は Factory によって固定されます。
- 境界: 汎用的な Markdown のケイパビリティ（構文サポート、パフォーマンス）はビジネスの変更ではありません。
- 例外と承認者: 既定ではありません。例外は対応するドメインの専門家が確認します。

既存の単一シートに登録し、標準コンテンツを再レンダリングする代わりに公開された Factory を再利用するコントリビューション:

```tsx
// The business contributes its own body into the seat the host already renders.
ctx.slots.inject('conversation.chat.reasoning.body', () => ctx.slots.register(
  { name: 'conversation.chat.reasoning.body', scope: 'session', locale: NS },
  TranslatedReasoningBody,
))

// The contribution keeps the standard rendering by calling the published Factory.
function TranslatedReasoningBody({ text, running, renderFactorySlot, t }: ReasoningBodyProps) {
  return (
    <>
      <p>{t('reasoning.translated')}</p>
      {renderFactorySlot('conversation.chat.reasoning.content', { text, running })}
    </>
  )
}
```

共有 Markdown パイプラインの内部にあるビジネス分岐:

```tsx
export const AssistantMarkdown = memo(function AssistantMarkdown({ blocks, ...props }: AssistantMarkdownProps) {
  return <>{blocks.map(block => (
    isHandoffBlock(block)
      ? <HandoffBlock block={block} />   // a business special case in the pipeline
      : renderBlock(block, props)
  ))}</>
})
```

すべてのビジネスの特殊ケースが 1 つの共有パイプラインに着地し、Markdown レンダリングは、それが運ぶようになったビジネスなしには変更も測定もできなくなります。

## ユーザー・メッセージ・モデルとレンダリングはビジネス・フィールドを運ばない

`UserMessage` と UI の `user` kind のレンダリングはインフラストラクチャです。メッセージ・モデルにビジネス専用フィールドはなく、ユーザー・メッセージ・レンダラーにビジネス分岐はありません。ビジネスの違いはメッセージ・ペイロード（自身の宣言マージで拡張可能なソース kind またはコンテンツ・フィールド）を通じて、あるいは登録されたレンダラーを通じて伝わります。1 つのビジネスのために追加されたフィールドはモデルのすべてのコンシューマーが処理し、次のビジネスは同じ共有レンダラーに別の分岐を追加します。

- 判定条件: `UserMessage` 上のビジネス名のフィールド、または `user` kind のレンダリング内のビジネス条件。
- 正しい出口: ペイロードでの表現。`MessageSourceMap` は宣言マージで拡張可能なため、プロデューサーは `UserMessage` を広げるのではなく自身のソース kind を宣言します。メッセージ・モデルは [session.md](../subsystems/session.md) に文書化されています。
- 境界: 汎用的なメッセージのケイパビリティ（添付、引用、編集状態）はビジネスの変更ではありません。
- 例外と承認者: 既定ではありません。例外は対応するドメインの専門家が確認します。

自身のソース kind を宣言するプロデューサーは、ビジネスの違いをペイロードに保ちます:

```ts ignore-check
declare module '@deepseek-ai/dsh-llm' {
  interface MessageSourceMap {
    handoff: { kind: 'handoff'; sessionId: SessionId }
  }
}

const message = createUserMessage({
  role: 'user',
  content: [{ type: 'text', text: prompt }],
  source: { kind: 'handoff', sessionId: source.id },
})
```

共有モデル上のビジネス・フィールドと、それが共有レンダラーに強いる分岐:

```ts ignore-check
interface UserMessage extends MessageBase {
  readonly role: 'user'
  readonly handoffKind?: 'handoff' | 'rag'   // one business's field on the shared model
}

// ...and the renderer branches on it.
if (message.handoffKind === 'handoff') return <HandoffBubble message={message} />
```

メッセージ・モデルのすべてのコンシューマーが 1 つのビジネスのフィールドを処理し、共有レンダラーは後に続くビジネスごとに分岐を増やします。
