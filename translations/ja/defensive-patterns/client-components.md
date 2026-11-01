# Client コンポーネントのルール

日本語 | [English](../../../docs/defensive-patterns/client-components.md) | [中文](../../../docs/defensive-patterns/client-components.zh.md)

> これは非公式の日本語訳です。英語版（[docs/defensive-patterns/client-components.md](../../../docs/defensive-patterns/client-components.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

クライアント・コンポーネント、その状態、およびそのエフェクトに関するルールです。[packages/client/AGENTS.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/AGENTS.md) が詳細な規律を担い、[slots.md](../subsystems/slots.md) が Slot の設計を、[web-styling.md](../web-styling.md) がスタイリングのルールを所有します。以下の各節は、要件、レビュアーがそれをどう判定するか、境界と例外がどこにあるか、そして正しい例と誤った例を示します。

## 状態は Injection を通じて届く

Slot のコンポーネントは、自身の登録の inject フェースを通じて、レンダリングするすべてのものを受け取ります。すなわち、プレーンなデータ、アクション・コールバック、そしてレンダラーが `use<Name>` フックに束縛する予約済みの `hooks` 区画です。一方、親のレンダー・サイトは、派生した `renderSlot` を通じてオーナー・データを渡します。ツリーの最上位で 1 つの大きなグローバル状態を読んで値を配ることはありません。すべてのライブ値をそのフェースに保つことが、変更されていない prop の同一性を保ち、コンポーネントが自身が読むサブ状態だけを購読できるようにします。これがなければ、無関係なフィールドの変更がサブツリー全体を再レンダリングし、ストリーミング・イベント下の長い Session で、ユーザーがスクロールや入力をする間にフレームを落とす原因になります。

- 判定方法: 登録時とレンダー・サイトで、著者は、レンダリングされるコンポーネントがどのサブ状態を購読し、受け取る prop のうちどれがレンダー間で安定した同一性を保つかを挙げられなければなりません。すべての prop の可変性に注意します。各レンダーで再構築される値（新しいオブジェクト、配列、インライン・コールバック）は、中身が変わっていなくても再レンダーを伝播します。
- 対象外: コンポーネント自身のローカル状態（折りたたみ、ホバー、下書きのための `useState`）は注入コンテンツではなく、React の外で何も購読しないコンポーネントに新しいシートは不要です。
- 例外: 新しいフック・シートは Slot の契約を変えるため、その変更はレビューで影響（どのコンポーネントが何を購読するか）を明示します。既存のフェースを使うだけなら、コンポーネントごとの承認は不要です。
- 修正方法: 注入をビューがレンダリングするものに絞り、読み取りごとに 1 つのフィールドをフック・シート経由で公開し、store やサービス・ハンドルではなく狭いアクション・セットを渡します。[コンポーネント入力](../subsystems/slots.md#component-inputs) が、inject ファクトリー、標準のフック・シート、およびそれぞれが prop になる方法を所有します。

登録はプレーンなデータ、狭いアクション・セット、1 つのフック・シートを運び、レンダー・サイトがオーナー・データを渡し、コンポーネントは自身がレンダリングするフィールドだけを読みます:

```tsx
// The inject factory returns plain data and callbacks; hooks is the reserved compartment.
ctx.slots.register(
  {
    name: 'sidebar.right',
    inject: () => ({
      rename,
      hooks: { info: infoSource },   // bound by the renderer as useInfo()
    }),
  },
  SidebarRight,
)

// The parent's render site passes owner data through the derived renderSlot.
{renderSlot('sidebar.right', { id })}

function SidebarRight({ id, useInfo, rename }: Props) {
  const { title } = useInfo()   // subscribes to title only
  return <button onClick={() => rename(id)}>{title}</button>
}
```

最上位のコンポーネントがビュー store 全体を読み、1 つの prop として配るため、レンダー・サイトは上記のどちらの問いにも答えられなくなります:

```tsx
function Sidebar() {
  const view = useViewStore()           // one top-level global read
  return <SidebarRight view={view} />   // the whole store travels down
}

function SidebarRight({ view }: { view: ViewStore }) {
  return <span>{view.activePanel.title}</span>
}
```

その store のすべてのフィールド変更がサイドバーのサブツリー全体を再レンダリングし、コンポーネントの購読集合はそれを束縛したサイトから見えなくなります。

## Props はデータとコールバックを運び、React ノードは運ばない

prop はデータまたはコールバックであり、React ノードは宣言された Slot を通じて渡ります。要素を prop の値や注入メンバーとして渡すと、Slot の契約が記述しない第 2 のコンポジション経路が隠れ、レンダー制御が登録サーフェスから呼び出しサイトへ移り、あるパッケージが別のパッケージのマークアップを組み立てられるようになります。ノード値の prop や注入メンバーは新たに追加しません。ホストが提供するマークアップが必要な箇所は、代わりに子 Slot として宣言します。

- 判定方法: コンポーネントの props 型、inject ファクトリーの戻り値、またはレンダー・サイトのオーナー props オブジェクトにある `ReactNode`、`ReactElement`、`() => JSX.Element`。
- 対象外: すでに出荷済みのノード値フィールドはそのまま動作します。このルールは、変更がそれに触れたときに発動し、その変更は拡張ではなく変換を行います。
- 例外: 同じ変更で変換できない場合、所有ドメインの専門家が延期を確認し、その変更は確認を記録します。著者が自己承認することはありません。
- 修正方法: 親の登録の `children` でその位置を宣言し、`renderSlot` でレンダリングし、子の props を通じてデータを渡します。[拡張ルール](../subsystems/slots.md#extension-rules) が、どのパッケージがどの Slot を宣言してレンダリングできるかを所有します。

ヘッダーの位置は宣言された子 Slot であるため、ホストはデータを渡し、登録されたコンポーネントが自身のマークアップを所有します:

```tsx
// The card declares the header location; the host passes data only.
ctx.slots.register(
  {
    name: 'tool.call.toolview',
    children: { header: { kind: 'single', scope: 'session' } },
  },
  ToolCard,
)

function ToolCard({ title, renderSlot }: Props) {
  return <article>{renderSlot('header', { title })}</article>
}
```

React ノードが prop の値として境界を越えるため、それが通るコンポジション経路はどこにも宣言されません:

```tsx
type ToolCardProps = {
  header?: React.ReactNode
  footer?: React.ReactNode
}

// The call site assembles another package's markup.
<ToolCard header={<ArchiveBadge />} />
```

カードの Slot 契約は何をレンダリングするかを記述しなくなり、呼び出しサイトがレンダー制御を保持し、バッジのパッケージはそれを渡す側からの実行時参照になります。

## Slot の登録は安定した関数コンポーネントである

Slot に登録される値は、モジュール・スコープで宣言された安定した関数コンポーネントです。レンダー中に作成されたコンポーネント関数はパスごとに新しいコンポーネント型になり、React は型の変更を別のコンポーネントとして扱います。サブツリーをアンマウントして新しいものをマウントするため、展開された行、入力フォーカス、スクロール位置がリセットされます。登録時のパラメーターは、レンダー時の変数をクローズ・オーバーするのではなく、props を通じてコンポーネントに届きます。

- 判定方法: 登録される値がどこで作成されるか。登録やレンダー呼び出しの中のインライン矢印関数、レンダー本体の内部で定義されたコンポーネント、インライン・コンポーネントを包む `useCallback` はすべて不合格で、モジュール・スコープの `function` 宣言は合格です。
- 対象外: このルールが制約するのは登録される値の同一性であり、その複雑さではありません。安定したコンポーネントは、注入された props、フック、コールバックを引き続き読めます。
- 例外: レンダー中にコンポーネントを作成するのが正しいのは、呼び出し元が再マウントを望む場合だけであり、通常の変更に別途の承認は不要です。Slot 契約の形式の変更は、「状態は Injection を通じて届く」で述べる inject フェースのレビューに従います。
- 修正方法: コンポーネントをモジュール・スコープへ移し、登録の shares から props の型を付け、変わる値をクロージャではなく inject フェース経由で渡します。

登録はモジュール・スコープのコンポーネントへの安定した参照を 1 つ持ち、変わる値は注入されて届きます。レンダー・サイトはローカル・コンポーネントを提供します:

```tsx
function PanelTitle({ title, count }: Props) {
  return <span>{title} ({count})</span>
}

// Registration: the local slot's scope is declared, never a component.
ctx.slots.registerFactory(
  {
    name: 'panel',
    scope: 'root',
    slots: { title: { scope: 'root' } },
    inject: () => ({ title: 'Sessions', count: 3 }),
  },
  Panel,
)

// The render site passes the local component, keeping its identity stable.
renderFactorySlot('panel', { /* owner data */ }, { slots: { title: PanelTitle } })
```

レンダー・サイトのインライン矢印関数は、そのレンダーが実行されるたびに新しいコンポーネント型を作成します:

```tsx
// A new component type each time this render runs.
renderFactorySlot('panel', { /* owner data */ }, { slots: { title: (p) => <span>{p.title}</span> } })
```

React は前回の登録とは異なる型を見てシートを再マウントし、それとは無関係の更新でローカル状態（展開、フォーカス、スクロール位置）をリセットします。

## コンポーネントは Cordis コンテキストを受け取らない

`ctx` は apply の世界に属します。プラグイン本体と、それをクローズ・オーバーする `inject` ファクトリーです。コンポーネントは Cordis コンテキストを決して受け取らず、サービス・クラスを import して操作することも、React コンテキストを読むこともありません。ビジネス・コンポーネントはコンテキストを 0 個しか見ないからです。レンダラーの束縛コンテキストはレンダラー内部のものです。裸の observable は、Injection がフックでラップした後にだけコンポーネントに届きます。直接購読するコンポーネントは、Injection フェース上の何も解放しないライフサイクルを所有します。

- 判定方法: コンポーネントのパラメーターや props 型にある `ctx` または `Context`。コンポーネント本体での `observable.value` の読み取り、`subscribe` の呼び出し、`useSyncExternalStore` の配線。機能の `.tsx` モジュールへ import されたサービス・クラス。
- 対象外: React の外で何も購読しないコンポーネント内部の振る舞いフック（タイマー、メディアクエリ）は注入コンテンツではなく、コンポーネント内に残ります。
- 例外: ありません。`TabSlot` に残っている直接購読は、そのエントリーの所有者が所有する是正作業であり、新しいコードの前例ではありません。
- 修正方法: その事実を登録の `inject`（プレーンなデータとコールバック、裸の observable は予約済みの `hooks` 区画）または宣言された store へ移し、派生したシートを通じて読みます。

Injection が observable を束縛し、コンポーネントはレンダラーが購読を所有する派生フックを 1 つ読みます:

```tsx
// The inject factory is the only place that closes over ctx.
const inject = () => ({
  hooks: { tab: tabStore.observable },
  action: { openTab },
})

function TabView({ useTab, action }: Props) {
  const { loaded } = useTab()   // the renderer owns the subscription
  return <button hidden={loaded} onClick={() => action.openTab('files')} />
}
```

コンポーネントが `ctx` を保持してソースを直接読むため、購読とその寿命はコンポーネントに属します:

```tsx
function TabView({ ctx }: { ctx: Context }) {
  const store = ctx.get('tabStore')                     // component holds the container
  return <span>{store.observable.value.title}</span>    // raw observable read in render
}
```

その購読を解放するものはなく、再レンダーは React のスケジューリングの外で発火し、コンポーネントは 1 つの store 実装に束縛されます。

## コンテキストは現在のスコープから取得する

`session`、`panel`、`tab` などは、現在のスコープ（Provider、注入メンバー、またはフレームワークのフック）から取得します。新しいグローバルな `current`、`active`、`selection`、`viewId` は、それらに到達するためのチャネルではありません。そのため、1 つの Session の複数のコンポーザーと会話はそれぞれ独立に動作します。

判定される条件は、コンポーネントがスコープを通じてコンテキストを取得するか、グローバル・フィールドを通じて取得するかです。マルチインスタンスの振る舞いがテストになります。1 つの Session 下の 2 つのコンポーザーが交差してはいけません。オブジェクトが本当にグローバルに一意である場合、サービスまたはインスタンスがそれを運び、呼び出し元が明示的に `inject` します。

```ts
// The injected factory closes over the scope's own session and panel.
interface Scope { readonly session: string; readonly panel: string }
const inject = (ctx: Scope) => ({ session: ctx.session, panel: ctx.panel })
```

```ts ignore-check
const activeTabId = globalThis.__activeTabId   // every tab reads the same value
```

誤った形式はインスタンス同士を混線させ、状態の所有関係を監査できなくします。

## リソースのコンシューマーは `useResource` を通る

リソースを表示する要素は、注入された `useResource` フックを通じて、リソースのプロトコルとアドレスで指定して読み取ります。ドメイン・ラッパー（Tab、パネル）は `useResource` の上に独自のフックを組み合わせ、その戻り値の構造は自由ですが、その下のフェッチ、キャッシュ、購読は作り直しません。プロトコルごとに 2 つ目の読み取り経路があると、キャッシュと無効化のポリシーが分裂し、1 つのアドレスが一方ではライブ、他方では古い状態になり得ます。

- 判定方法: 読み取り経路。`useResource<Protocol>(address)`、またはその本体でそれを呼ぶビジネス・フックは合格です。手作りの購読とキャッシュ、プロバイダーのストリーム処理のコピー、`useTabStream` のような平坦な汎用フックは不合格です。
- 対象外: リソースではないローカル UI 状態（折りたたみ、ホバー、下書き）は `useState` のままで、リソース・アドレスを持ちません。[Client リソース](../subsystems/client-resources.md) がステータス値と、リソースを開いたままにするものを所有します。
- 例外: スナップショットの再形成は自由ですが、条件分岐の背後にプロトコルの違いを隠すラッパーはレビューで所有関係を明示し、新しいプロトコルのフックはそのプロトコルの所有者から提供されます。
- 修正方法: ドメイン・フックの内部で `useResource` を呼び、その結果をコンポーネントがレンダリングするものへ対応付け、特注のキャッシュを削除します。

ドメイン・ラッパーは注入されたフックを引数として受け取り、その結果を再形成するだけなので、最下層は 1 つのままです:

```tsx
// The bottom layer is the standard hook, reached through the component's inject face.
function useChatMessages(useResource: UseResource, id: string) {
  const resource = useResource<'chat'>(`dsh-resource://chat/${id}`)
  return resource.status === 'live' && resource.value !== undefined
    ? { messages: resource.value.messages, loading: false }
    : { messages: [], loading: true }
}

function ChatBody({ id, useResource }: Props) {
  const { messages } = useChatMessages(useResource, id)
  return <ol>{messages.map((message) => <li key={message.id}>{message.text}</li>)}</ol>
}
```

ラッパーが購読、キャッシュ、無効化を自身の名前で作り直すため、プロバイダーのライフサイクルと共有キャッシュはもはや適用されません:

```tsx
export function useTabStream(id: string) {
  const [value, setValue] = useState<TabData>()            // bespoke cache
  useEffect(() => {
    const subscription = tabProtocol.open(id).subscribe(setValue)   // bespoke subscription
    return () => subscription.close()                              // bespoke teardown
  }, [id])
  return value
}
```

すべてのプロトコルが独自のフェッチと無効化のロジックを増やし、キャッシュ・エントリーと更新ポリシーを統一できず、1 つのアドレスのコンシューマー同士がその現在値について食い違います。

## store の使用は狭く明示的である

フロントエンドの store は、共有の表示状態または操作状態を保持します。その spec は、初期状態、それを変更するアクション、そして値がリロードを生き延びるべき場合にはフレームワークが機械的に実行する永続化キーを宣言します。Session スコープのインスタンスはそのキーに接尾辞を付け、Session ごとのインスタンスが独立に永続化されるようにします。store の属性とそれらを変更するアクションはカプセル化され、狭いインターフェースとして注入されるため、コンポーネントはそれらと必要なセレクターを受け取り、ビュー store 全体を受け取ることはありません。ビジネス・データは store ではなくオブジェクト層に存在し、設定の変更は store の状態ではなく設定インターフェースを通ります。

- 判定方法: 注入されるフェースが狭いインターフェースか store オブジェクトそのものか、またハンドルが `apply` の内部で構築されるかモジュール・レベルで構築されるか。
- 永続化: spec の `persist` キーがメカニズムのすべてであり、フレームワークが実行します。エンジン・インスタンスは `getSnapshot`、`subscribe`、焼き込まれたアクション、`clearPersisted` を公開し、何も手作業で水和しません。
- 対象外: store 内のレンダー状態は設定を変えない限り問題なく、コンポーネント・ローカルの状態は store のコンテンツではありません。[store の契約](../subsystems/slots.md#component-inputs) が、宣言された store がどう props になるかを所有し、[Web Client アーキテクチャ](../subsystems/web-client.md) がビジネス・データの置き場所を所有します。
- 例外: モジュール・レベルでエクスポートされた store ハンドルは禁止です。モジュール・キャッシュの同一性は、プラグインのリロードをまたぐ偽装されたシングルトンです。注入フェースを狭いものへ再構成するのに別途の承認は不要です。
- 修正方法: inject の戻り値にある store ハンドルを、コンポーネントがレンダリングするフィールドとアクション・コールバックに置き換え、ハンドルを `apply` の内側に保ちます。

1 つのハンドルが `apply` の内部で構築されて登録に渡され、inject フェースはビューがレンダリングするフィールドとアクションだけを運びます:

```tsx
// The handle is built in apply; persistence is the spec's own key.
const sidebarStore = createSidebarStore()   // defineStore({ init, actions, persist: 'sidebar' })

ctx.slots.register(
  { name: 'sidebar.header', store: sidebarStore, inject: () => ({ onSelect: selectPanel }) },
  SidebarHeader,
)

// The component reads through the framework-bound seat and writes through the actions.
function SidebarHeader({ useStore, actions, onSelect }: Props) {
  const { title } = useStore()
  return <button onClick={onSelect}>{title}</button>
}
```

ビュー store 全体が注入され、コンポーネントがそこからフィールドを選び、モジュール・レベルのハンドルがインスタンスのライフサイクルをフレームワークから奪います:

```tsx
// The component receives the store itself and picks its own fields.
const inject = () => ({ viewStore })

function SidebarHeader({ viewStore }: Props) {
  return <button onClick={viewStore.select}>{viewStore.activePanel.title}</button>
}

// Module-level handle: one identity shared by every plugin load.
export const paneStore = createPaneStore()
```

コンポーネントの依存フェースは監査できなくなり、共有モジュールの同一性はすべてのスコープ破棄を超えて生き残ります。2 つのペインはそれぞれ 1 インスタンスではなく、1 つのインスタンスを共有することになります。

## 同一性と初期状態は意図的に決める

状態の切り替えはコントロールを再マウントせず、その DOM ノードを置き換えません。インスタンスとその DOM の同一性は状態変更をまたいで同じままなので、フォーカス、選択、進行中の下書きが生き延びます。初期状態は最初のフローが実行されるときに準備されます。同じ経路がデータを準備して状態を確立し、後からエフェクトで埋めることはありません。コンポーネントはサービス全体ではなく、フック、セレクター、アクション・コールバックを受け取ります。

- 判定方法: 状態変更後もコントロールのインスタンスとその DOM ノードが同一性を保つか。アンマウントしてからマウントすることが違反です。初期化経路も同じように判定します。最初のフローの内部で確立された状態は合格し、エフェクトから書き込まれた状態は不合格です。
- 依存フェース: フックが読み取り、セレクターがサブ状態を選び、アクション・コールバックが書き戻します。サービス全体を prop や注入メンバーとして渡すことは不合格です。
- スコープ: 内部インスタンスとローカル状態を保持する入力コントロールとコマンド・コントロール、およびファクトリーが作成するビジネス・ビュー。
- 例外: 真に必要な再マウント（インスタンス状態を移行できないデータ・セットへの切り替え）は、状態移行の計画とともにレビューで説明します。コントロールのインスタンス寿命に影響する変更のレビュアーがそれを受け入れます。
- 修正方法: 最初のフローが状態を構築する場所で初期値を導出し、レイアウト・ノードをマウントしたまま props を変化させ、サービス prop を 3 つの狭い shares に置き換えます。

下書きは Session とともに存在し、コンポーザー・ノードはマウントされたまま、その props だけが変わります:

```tsx
// The draft is built by the first flow; nothing fills it in later.
const draft = useDraftState(sessionId)
const { canSend } = useComposerSelectors(draft)

// The node stays mounted; state changes reach it as props.
return (
  <div className={s.composer} data-ready={canSend || undefined}>
    {draft.text}
  </div>
)
```

初期状態が 1 コミット遅れて届き、キー付きのコントロールが Session の変更のたびに再構築されます:

```tsx
// The draft is filled in after the first commit.
useEffect(() => { setDraftState(initialDraft) }, [])

// The keyed composer unmounts on every session switch.
<Composer service={composerService} key={sessionId} />
```

最初のフレームは空の下書きをレンダリングし、Session の切り替えのたびにフォーカス、選択、ユーザーが入力していた内容を失います。

## `useLayoutEffect` は既定ではない

`useLayoutEffect` は既定では使わず、`useEffect` に置き換えることも修正ではありません。2 つはコードが何をしているかではなく、タイミングが異なるからです。レイアウト測定は、まずなぜここで行うのか、なぜ CSS で表現できないのか、なぜピクセル測定でなければならないのかに答えます。その後でだけ、所有ドメインの専門家の承認を得て、注入された測定フックへ移ります。どちらの種類のエフェクトも状態を記録したりロジックを束縛したりしません。それは Injection に属します。

- フック名ではなく目的で判定します。エフェクトが何の測定のため、または何の束縛のために存在するかです。状態の記録やロジックの束縛を行うエフェクトは Provider または注入されたフックとして Injection に戻ります。3 つの問いを生き残った測定だけが残り、その結果は宣言的なレイアウト・パラメーターとして消費されます。
- 対象外: 2 つのエフェクトのタイミングの違いは、`@types/react` が `useLayoutEffect` を DOM 変更後かつペイント前に同期的に走ると文書化している背景的事実であり、一方を他方に書き換える許諾ではありません。
- 例外: 残す `useLayoutEffect` には所有ドメインの専門家の確認が必要で、その変更は確認を記録します。
- 修正方法: レイアウトを CSS（grid、`aspect-ratio`、コンテナ・クエリ）で表現するか、注入されたフックの内部で測定を集めて 1 つの値を公開します。

レイアウトは CSS で宣言され、問いを生き残った 1 つの測定が注入されたフックに残ります:

```tsx
// Fix 1 — express the layout in CSS and delete the measurement.
// .card { display: grid; grid-template-columns: minmax(0, 1fr) auto; aspect-ratio: 16 / 9; }

// Fix 2 — the approved measurement is injected and returns one value.
function CardBase({ useLayoutMeasurements }: Props) {
  const ref = useRef<HTMLDivElement>(null)
  const { gutterWidth } = useLayoutMeasurements(ref)
  return <div ref={ref} className={s.card} style={{ '--gutter': `${gutterWidth}px` }} />
}
```

下の最初の 2 つのエフェクトはどちらも間違った問い（どのエフェクトを使うか）に答えており、3 つ目は束縛をエフェクトに置き去りにしています:

```tsx
// A synchronous pixel measurement the design did not need.
useLayoutEffect(() => { setHeight(ref.current.getBoundingClientRect().height) }, [])

// The same measurement with the hook name swapped.
useEffect(() => { setHeight(ref.current.getBoundingClientRect().height) }, [])

// Logic binding parked in an effect instead of Injection.
useEffect(() => { bindShortcut(handler) }, [])
```

名前を変えてもレイアウトの問題は残り、測定された高さはどのエフェクトを選んだかに依存するようになり、ショートカットは Injection を通るのではなく 1 レンダー遅れて束縛されます。

## エフェクトはレンダー由来のデータを親に push しない

エフェクト本体は、コンポーネントがすでに持つ props や状態からレンダー中に決められるデータを、props や Slot props を通じて届いたセッターで親に push しません。そのような push は 2 度目のレンダーを要し、親の最初のフレームに値がなく、その後の状態変更をレンダー・カスケードに変えます。値はレンダー中に決定され、ホストがレンダー時のチャネルを通じて受け取るか、ユーザー・ジェスチャーが自身のハンドラーで自身の状態を更新します。

- 判定方法: props や Slot props のセッターがエフェクトの内部で呼ばれ、その引数が現在の props または状態から派生し、外部イベントや購読コールバックがそれを駆動していないこと。クリーンアップの `return () => setX(undefined)` も同じ違反の一部です。
- レビューで挙げる帰結: 余分なレンダー・パス。親の最初のフレームに値がないこと。メンバーを省略すると古くなる依存配列。エフェクト → 親の状態 → レンダーのカスケード。親の状態に絡みつく子のライフサイクル。
- 対象外: 真の外部イベント（購読コールバック、ソケット・フレーム、ユーザー・ジェスチャー）に反応するエフェクトはこのパターンではなく、イベント・ハンドラーは状態を直接更新してかまいません。
- 例外: 既定ではありません。例外には所有ドメインの専門家の確認が必要です。
- 修正方法（順に）: 状態がすでに存在する場所で UI をレンダリングする。ホストがレンダリングしなければならない場合は、レンダー中に値を渡す。状態がエントリー間で共有される場合は、inject フェースまたは宣言された store へ移す。

コントロールはその状態がすでに存在する場所でレンダリングされ、共有バリアントはレンダー中に読まれ、書き込みはジェスチャー・ハンドラーに残ります:

```tsx
// 1. Render it where the state already lives; no channel is needed.
return <Button onClick={() => setShowOriginal((v) => !v)}>{label}</Button>

// 2. The host renders it: the flag is declared state, read during render
//    and written only from the gesture handler.
const { showOriginal } = useStore()
const { toggleOriginal } = actions
return <Button onClick={toggleOriginal}>{showOriginal ? rendered : source}</Button>
```

`label` と `disabled` はレンダーから導出できるため、エフェクトから親のセッターを呼ぶと、親は 2 度レンダリングし、それでも最初のフレームにヘッダーがありません:

```tsx
useEffect(() => {
  setHeaderAction({ label, disabled, onClick })   // render-derivable data pushed up
  return () => { setHeaderAction(undefined) }
}, [label, disabled, setHeaderAction])
```

親は最初のパスで持てたはずのデータのために 2 度レンダリングし、ヘッダーは 1 フレーム遅れて現れるか点滅し、手書きの依存リストがそれをいつ公開するかを決めます。

## ビルドとスタイルは所有者に従う

クライアント・パッケージのすべてのビルド調整は、そのパッケージの tsdown プラグインを通り、パッケージが独自のビルド分岐を増やすことはありません。スタイリングは CSS Modules を通り、インライン化されるのは画像、GIF、SVG だけです。しかも JSX ではなく CSS にインライン化されます。大きなテキストが TSX にインラインで現れることはなく、`style` オブジェクトは実行時に計算される値だけを保持します。静的な値とスタイル・バリアントは CSS Module のクラス名に属します。

- 判定方法: パッケージ内の新しいビルド分岐（tsdown プラグインの代わり）、機能スタイリングでのリテラルの色、コンポーネント・ライブラリ、Tailwind。TSX の base64 データまたは大きなテキスト。静的値やバリアントの切り替えを運ぶ `style` オブジェクト。
- 対象外: 測定されたガターや進捗率のような実行時計算値は `style` に入ってよく、コンポーネント・ローカルのカスタム・プロパティがレイアウト値を運んでもかまいません。[コンポーネントのルール](../web-styling.md#component-rules) がトークン、CSS Module、クラス名のルールを所有します。
- 例外: 新しいビルド分岐や新しいインライン形式は、着地前にレビューで説明され、所有パッケージのルールがそれを残すかどうかを決めます。
- 修正方法: バリアントを `className` で切り替える CSS Module のクラスへ移し、base64 をビルド・チェーンが生成する CSS に保ち、大きなテキスト定数を TSX モジュールから削除します。

コンポーネントはクラス名とデータ属性を切り替え、CSS Module がプレゼンテーションを所有します:

```tsx
import s from './card.module.css'

// Style variants are class names, not style-object branches.
export function Card({ active, tone, children }: Props) {
  return (
    <div className={clsx(s.card, active && s.active)} data-tone={tone}>
      {children}
    </div>
  )
}
```

大きなテキストとインライン化されたバイナリ・データがモジュールに入り、`style` オブジェクトがテーマで上書きできないバリアントをエンコードします:

```tsx
const HUGE = '...several KB of text...'   // large text inline in the module

// A variant switch and an inlined asset both sit in the style object.
<div style={{ color: toneColor, backgroundImage: `url(${AVATAR_DATA_URL})` }} />
```

バンドルは CSS とアセットに属すべきテキストとバイナリ・データを運び、diff は読めないブロブを運び、テーマの上書きはその色にもはや届きません。

## key は何を識別するかを示す

`key` を渡すのは、要素がどのエンティティまたはインスタンスを表すかを示す場合だけにし、その同一性を明示します。たとえば `turnId` は 1 つの Session 内で一意です。ほとんどのリストはすでに安定したビジネス id を持ち、余分な key を必要としません。警告を黙らせるためやレンダー順をエンコードするために発明された key は、データが持たない同一性を示します。配列インデックスがその同一性になることは決してありません。挿入や削除はそれ以降のすべての項目を別の要素へずらすからです。

- 判定方法: 項目の同一性を位置で表現できるか、安定した id で表現できるか。key が渡される場合、変更は何がそれを一意にするかを明示し、可変リストに対する `key={index}` は不合格です。
- 対象外: リストの項目が key を持つという React 自身の要件は引き続き適用されます。その key は項目の安定したビジネス id であり、その隣の 2 つ目の key ではありません。
- 例外: ありません。通常の変更に承認は不要ですが、共有コンポーネントの key のセマンティクスはレビューで明示されます。
- 修正方法: エンティティの安定した id を使い、2 つの行が本当に 1 つの id を共有する場合は、key にインデックスを混ぜるのではなく id のソースを修正します。

key は turn 自身の id であり、Session 内でのその一意性が要素の表す同一性です:

```tsx
// turnId is unique inside a session; the id is the identity.
export function TurnList({ turns }: Props) {
  return (
    <ol>
      {turns.map((turn) => <TurnRow key={turn.turnId} turn={turn} />)}
    </ol>
  )
}
```

インデックスが同一性の代わりを務めるため、リストが変わるとそれ以降のすべての行が同一性を変えます:

```tsx
// The index stands in for identity.
{items.map((item, index) => (
  <Row key={index} data={item} />
))}
```

挿入や削除の後、それ以降の各行は新しいデータに対して以前の状態を保つため、展開された行や進行中の編集が誤った項目に現れます。

## 各共有レイヤーは独自の分離レベルを保つ

機能は自身の要件を自身のパッケージで吸収し、共有コードからどれだけ分離を保つべきかは、そのコードがどの共有レイヤーに属するかで決まります。探すべき違反は、機能固有の概念が共有レイヤーに置かれることです。それにより他のすべてのコンシューマーが自分たちのものではない概念を理解する必要が生じ、その分岐を撤回するのが難しくなります。

| 共有コード | 機能が保つ分離 |
|---|---|
| `ui-primitives`、カタログ化されたコントロール | カタログ化されたコントロールを消費します。新しいコントロールは機能パッケージで始まり、2 番目のコンシューマーが必要としたときにだけ共有カタログへ昇格します。 |
| `ui-layout` とシェル・レイアウト | それを消費します。機能パッケージ内でラッパー、上書きしたスタイル、または追加レイヤーによって適応します。 |
| `ui-renderer` とその他のレンダー機構 | ビジネス・コードからの値依存はありません。文書化された登録シーム（Slot、provide/inject に束縛されたフック、宣言された store）を通じてのみ拡張します。 |
| 別の機能プラグイン | その値を実行時に import せず、再エクスポートもしません。UI は Slot を通じて、振る舞いは注入されたサービスを通じて越えます。共有ランタイム・コードは `client/store`、`ui-primitives`、またはブラウザ安全なユーティリティ・パッケージのような狭い静的オーナーに置きます。 |
| データ・オブジェクト層（`client/connection`、`session-controller/client`、`client/store`） | 文書化されたサーフェスを通じて読み取ります。機能独自の概念は、これらのオブジェクトやその store に入りません。 |
| 機能自身のパッケージ | 無制限です。要件はここで吸収され、独自の行、順序付け、プレゼンテーションを含みます。 |

- 判定方法: diff が機能固有の分岐、フィールド、概念を、機能自身のパッケージ以外のレイヤーに置いていないか。
- 対象外: 機能が所有するコード、および 1 つの機能の必要ではなく真に汎用的な共有レイヤーの変更。
- 例外: 共有レイヤーの変更は所有ドメインの専門家の確認を伴い、すべてのコンシューマーに対してリグレッションを確認します。GUI の変更は [packages/client/AGENTS.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/AGENTS.md) の最小チェックを実行します。
- 修正方法: 要件を機能パッケージで吸収するか、必要性が真に汎用的な場合は、所有者の承認を得て独自の変更で共有レイヤーへ昇格します。

機能は自身の行レンダリングと順序付けルールを所有し、基底コンポーネントは変更せずに消費されます:

```tsx
// The feature owns its rows; the base is consumed, not edited.
// packages/client/ui-workspace/src/client/rows/WorkspaceBrowser.tsx
export function WorkspaceBrowser({ rows, onArchive }: Props) {
  const ordered = [...rows].sort((a, b) => Number(b.archived) - Number(a.archived))
  return (
    <ul>
      {ordered.map((row) => <WorkspaceRow key={row.id} row={row} onArchive={onArchive} />)}
    </ul>
  )
}
```

基底コンポーネントが 1 つの機能の順序付けルールを学ぶため、すべてのコンシューマーが自分たちのものではない概念を継承します:

```tsx
// ui-layout: the base learns one feature's ordering rule.
export function BrowserList({ rows, ...rest }: Props) {
  const ordered = [...rows].sort((a, b) => Number(b.archived) - Number(a.archived))
  return <ul {...rest}>{ordered.map((row) => <Row key={row.id} row={row} />)}</ul>
}
```

`ui-layout` の他のすべてのコンシューマーは、1 つの機能だけが持つアーカイブの概念を理解しなければならなくなり、分岐の撤回にはそれらすべての監査が必要になります。
