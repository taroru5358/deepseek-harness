# Web Client スロット

日本語 | [English](../../../docs/subsystems/slots.md) | [中文](../../../docs/subsystems/slots.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/slots.md](../../../docs/subsystems/slots.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

スロットは Web Client の型付き React コンポジション・システムである。[`dsh-client-ui-slots`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-slots/README.md) は React 非依存のレジストリと型代数を定義し、[`dsh-client-ui-renderer`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-renderer/README.md) は observable ソースをフックに束縛してツリーをレンダリングし、React コンテキストを内部で所有する。フィーチャー・プラグインは `ctx.slots.register()` を通じて UI を提供し、他のフィーチャー・プラグインのコンポーネントをインポートすることはない。

このページでは、スロットの所有権、コンポーネントの入力、拡張 API、出荷される階層を説明する。周辺の boot、Remote、Client モデル、Conversation の経路は [Web Client アーキテクチャ](web-client.md) にある。

`plugins.bundle.config` は npm パッケージ名をキーとする bundle 詳細設定を提供する。`plugins.bundle.activation` はユーザーが要求した有効化の後に任意のガイダンスをレンダリングし、その bundle の詳細を閉じたり開いたりするオーナー・コールバックを持つ。`conversation.input.activity` はモデル・セレクターと Send の間に 1 つのアクションを提供し、ツールバーの展開はアンマウント時に解放される。

プラグイン・カタログは `plugins.add.actions` を宣言する。これは Add plugin メニューでのインストール後に `MenuItemButton` の行を並べる root スコープのリストである。そのオーナーは、アクションを開始する前にメニューを閉じるための `onDismiss()` を提供する。スロットは所有する Plugins ページの宣言ライフタイムに従う。

root スコープのリスト `sidebar.panellist.actions`、`conversation.session.header.views.actions`、`sidebar.right.tab.guide.actions` は、パネル行、View タブ、右サイドバーのガイド・カードの後の More メニューの行を提供する。リージョンは、そのリストに登録がある間だけ More ボタンとメニューを描画し、カスタマイズ可能なエントリーをピン状態、`onOpen`、`onPinnedChange`、`onDismiss` とともに渡す。パネルと View のエントリーは、ピン状態とは独立に、現在の選択に対する任意の `active` 状態も運ぶ。各リージョンはピンされていないエントリー id をメモリー内に保持する。[`ui-entry-customization`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-entry-customization/README.md) が出荷される行を登録する。

[ui-workspace](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-workspace/README.md#understand-the-implementation) は Session 行の拡張位置、可視性、キーボード挙動を説明する。

## 宣言とライフサイクル

`SlotMap` はコンパイル時のレジストリである。パッケージはキー、カーディナリティ、スコープ、オーナー props、キー付き props、任意のスロット・レベルの inject フェイスを宣言マージする。実行時の宣言は、レンダリング位置を所有するコンポーネント上の対応する `children` エントリーである。

子を宣言すると 3 つの効果がある。子キーが有効になり、その親エントリーの `renderSlot` または `renderSlotChain` 呼び出しが認可され、実行時の dispatch 仕様が記録される。宣言ごとに 1 つの有効なエントリーが所有する。未宣言のスロットへの登録や、他所で既に所有されている子の宣言は、プラグインのアクティベーション時に失敗する。

`root` は唯一の組み込み宣言であり、Cordis サービス自体を通じてレンダリングされる唯一のキーである。`ui-renderer` は `ctx.slots.renderSlot('root', {})` を呼ぶ。すべての子孫は、それを宣言したエントリーの `renderSlot` または `renderSlotChain` prop を通じてレンダリングされる。

登録と宣言は Cordis の effect ライフタイムに従う。エントリーを破棄するとそのコントリビューションが削除され、それが宣言した子スロットが再帰的に折りたたまれる。したがって、別パッケージのスロットにコントリビュートするフィーチャーは `ctx.slots.inject(key, callback)` を使う。コールバックは宣言のライフタイムごとに実行され、オーナーが折りたたまれるとその effect が削除され、オーナーが再びマウントされると再実行される。

```tsx ignore-check
import type { Context } from '@deepseek-ai/cordis'
import type {} from '@deepseek-ai/dsh-client-ui-conversation/client'
import type {} from '@deepseek-ai/dsh-client-ui-session/client'
import type { PropsRuntime } from '@deepseek-ai/dsh-client-ui-slots'

type HeaderActionProps = PropsRuntime<'conversation.session.header.actions'>

function HeaderAction({ useSession }: HeaderActionProps) {
  const running = useSession(snapshot => snapshot.running)
  return <button disabled={running}>Review</button>
}

export const inject = ['slots']

export function apply(ctx: Context): void {
  ctx.slots.inject('conversation.session.header.actions', () =>
    ctx.slots.register({
      name: 'conversation.session.header.actions',
      id: 'review',
      order: 100,
    }, HeaderAction))
}
```

## カーディナリティとスコープ

スロット宣言は 2 つの独立した軸を固定する。

| 軸 | 値 | 意味 |
|---|---|---|
| カーディナリティ | `single` | 1 つのセル。有効な優先度の勝者がレンダリングされる。これを追加型リストとして扱わず、子スロットを使うこと。 |
| カーディナリティ | `list` | セルは必須の `id` で指定され、`order`、次に登録順で並ぶ。 |
| カーディナリティ | `keyed` | オーナーが `entryKey` を dispatch し、一致するセルがキー固有の props とともにレンダリングされる。 |
| カーディナリティ | `chain` | 各エントリーが純粋な `select(owner)` 関数を提供する。優先度順で最初の非 null の結果がレンダリングされ、その結果を `matched` として受け取る。そうでなければオーナーのフォールバックがレンダリングされる。 |
| スコープ | `root` | root スコープのコンポーネントとストア・インスタンスが 1 つ。 |
| スコープ | `session-maybe` | 周囲の Provider 束縛を継承するが、束縛なしでもレンダリング可能。Session の値は任意。 |
| スコープ | `session` | 解決済みの周囲の Provider 束縛を必要とし、確定した Session の値を受け取る。 |

`priority` は `single`、`list`、`keyed` のセルではシャドーイングの順位、`chain` では選出順である。値が小さいほど先に実行またはレンダリングされる。通常の追加型コントリビューションは新しいリスト `id` またはキー付き `key` を選ぶべきである。出荷済みのセルを意図的に再利用すると、そのプレゼンテーションが置き換わる。

[コンテキストメニュー・プラグイン](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-context-menu/README.md)は `provideRoot` を通じて `useContextMenu` を提供する。すべてのスコープが同じプラグイン所有のトリガー・ファクトリーを受け取る。その prop と保持されたハンドラーは、プロバイダーのアンロード時に退役する。その root スコープのキー付き `contextMenu.icon` スロットは、アクションのメタデータからビジネス所有のアイコンをレンダリングする。

## コンポーネントの入力

登録されたコンポーネントは、その束縛箇所で組み立てられた入力を受け取る。コンポーネントはメンバーをコピーするのではなく、これらの型を導出する。

| 入力 | 宣言元 | コンポーネント型 |
|---|---|---|
| オーナーの値と標準スコープの値 | `SlotMap` の行とインストール済みのスコープ・アダプター | `PropsRuntime<K>` |
| 認可された子レンダラー | 登録の `children` キー | `PropsRenderSlots<S>` |
| 共有ビュー状態のセレクター・フックとミューテーション・コールバック | 登録の `store` | `PropsStore<H>` |
| プライベート・データ、コールバック、observable フック | 登録の `inject` ファクトリー | `InjectFace<I>` |
| ローカライズされた `t` 関数 | 登録の `locale` 名前空間 | `PropsLocale<N>` |
| 選択された chain の値 | 登録の `select` の結果 | `ComposedProps` を通じた `matched` |

エントリーが `session` または `session-maybe` の子を宣言する場合、`SessionProvider` は `PropsRenderSlots` にも存在する。`session` prop がない場合は周囲の束縛を継承し、明示的な `SessionReference` または `undefined` はそのサブツリーだけを上書きする。Provider はボディ全体をキー付けしない。厳密な `session` エントリーは、束縛の世代が変わると再マウントする。空の `session-maybe` エントリーは、再マウントせずに最初の束縛を採用し、その後の世代変化または不在への復帰で再マウントする。

コンポーネントが `ctx` を受け取ることはない。親が所有する時点値は `renderSlot` のオーナー引数を通じて入る。共有ビュー状態は宣言されたストアを使う。サービスとモデル・オブジェクトは `apply` のクロージャーに留まり、コールバックまたは observable ソースに投影される。

## フレームワーク提供のフック

出荷されるアダプターはこれらの標準 props を追加する。それらはコンポーネントを登録したパッケージに関係なく、対象スロットのスコープに応じて利用できる。

| 利用可否 | Props | オーナー |
|---|---|---|
| すべてのスコープ | `useSessions`, `useSessionStatus`, `useSessionRetainInfo`, `useSessionAccess` | `ui-session` |
| すべてのスコープ | `useWorkspaces`, `useWorkspaceAccess` | `ui-workspace` |
| すべてのスコープ | `useRemoteAccess`（`global` リソース） | `api-remote-access` |
| すべてのスコープ | `usePanelInfo` | `ui-layout` |
| `session` | `sessionId`, `useSession`, `useProjection`, `useAccess`（束縛された Session の回答） | `ui-session` |
| `session-maybe` | 任意の `sessionId`, `useSession`, `useProjection`, `useAccess` の結果 | `ui-session` |
| `session` | `useConversation`, `useInput`, `inputActions` | `ui-conversation` |
| `session-maybe` | 任意の `useConversation`, `useInput`, `inputActions` の結果 | `ui-conversation` |
| `session` | `useChat` | `ui-chat` |
| `session` | `useTrajectory` | `ui-trajectory` |

レンダラーはまた、宣言されたストアから `useStore` を、宣言されたロケール名前空間から `t` を生成する。これらはグローバル標準 props ではなく、登録から導出される props である。

フレームワークおよびドメイン・アダプターのオーナーは、`ctx.slots.provideRoot()` または `ctx.uiSession.provide()` を、対応する `GlobalStandardProps`、`SessionStandardProps`、`SessionMaybeStandardProps` の宣言マージとともに使い、標準セットを拡張できる。フィーチャー・コンポーネントは、自前で React フック prop を作ったり、エントリー固有のデータのためにグローバル標準 prop を追加したりすべきではない。

## 開発者提供のインジェクション

登録の `inject` オプションは、通常のフィーチャー所有のインジェクション・ポイントである。そのファクトリーはプラグインの `apply` ワールドで実行され、注入された Cordis サービスをクローズオーバーでき、コンポーネントが必要とするデータとコールバックだけを返す。`session` スロットでは `sessionId` を、`session-maybe` では `sessionId | undefined` を受け取る。ストアが宣言されている場合は、そのストアの束縛済みアクションも受け取る。

その戻り値の予約済み `hooks` オブジェクトは、素の `getSnapshot`/`subscribe` ソースを受け付ける。レンダラーは `hooks: { status }` を `useStatus(selector)` コンポーネント prop に変換し、束縛をソースのアイデンティティでキャッシュする。コンポーネントはソース自体を受け取らず、`useSyncExternalStore` を直接呼ぶこともない。

すべての占有者が同じケイパビリティを必要とする場合、スロットのオーナーは子宣言に `inject` フェイスを置ける。プレーンなメンバーはすべての占有者にそのまま届く。その `hooks` オブジェクト内の関数値メンバーはフック・ファクトリーであり、スロットの標準 props と任意のレンダーごとの `hookContext` を受け取り、占有者に公開される制約付きフックを返す。`conversation.chat.node` はこの仕組みを使って、現在レンダリング中のノードに対する `useTurnData(key)` を提供する。

1 回のレンダリング箇所で既に分かっている値にはオーナー props を、1 エントリーのコールバックとプライベートな observable には登録の `inject` を、スロット・オーナーが制御するケイパビリティにはスロット・レベルの `inject` を、エントリー間で共有される、または再マウントをまたいで保持される可変ビュー状態には宣言されたストアを使う。React ノードは注入値ではなく子スロットを通じて合成される。

<a id="current-hierarchy"></a>

## Conversation 階層

以下の Conversation 経路は、宣言オーナーと再利用可能な Factory ルートを区別する。無関係なシェル位置は省略する。エントリーは自身が宣言した子を所有し、Factory は登録された定義のライフタイムの間、その子宣言を所有する。角括弧の名前は既定の React コンポーネントを識別する。素の React 子孫はオーナーのスロットをレンダリングできる。Factory 呼び出しは独立した再利用可能な定義への参照であり、新しいスロット宣言ではない。

```text
root [AppFrame]
└─ main[key=conversation] [ConversationPanel]
   └─ main.conversation [ConversationRoot / ConversationMainPanel]
      ├─ conversation.header [ConversationHeader]
      │  ├─ conversation.header.leading
      │  └─ conversation.session.header [ConversationSessionHeader]
      │     ├─ conversation.session.header.lineage
      │     ├─ conversation.session.header.actions
      │     ├─ conversation.session.header.utilities
      │     ├─ conversation.session.header.views.actions
      │     └─ conversation.session.header.corner
      └─ calls Factory: conversation.content

Factory: conversation.content [ConversationContent]
├─ conversation.session [ConversationSession / DefaultConversationViews]
│  └─ conversation.view
│     ├─ id=chat [ChatView]
│     │  ├─ conversation.chat.text-interaction [list]
│     │  └─ conversation.chat.flow [ChatFlow]
│     │     └─ conversation.chat.node [ChatNodeSeat dispatch]
│     │        ├─ key=user [UserMessageNodeView]
│     │        │  ├─ conversation.chat.user.before [chain]
│     │        │  └─ calls Factory: conversation.chat.user.content
│     │        ├─ key=steering [UserMessageNodeView]
│     │        │  └─ calls Factory: conversation.chat.user.content
│     │        ├─ key=assistant-step [AssistantNodeView / AssistantMarkdown]
│     │        │  ├─ conversation.chat.message.images [MessageImages]
│     │        │  └─ conversation.chat.reasoning.body [DefaultReasoningBody]
│     │        │     └─ calls Factory: conversation.chat.reasoning.content
│     │        ├─ key=tool-call [ToolCallTree]
│     │        │  └─ tool.call.toolview
│     │        │     └─ key=read_image [ReadImageRow / ToolRow]
│     │        │        └─ tool.call.images [MessageImages]
│     │        ├─ key=command-input [GoalCommandInputView]
│     │        │  └─ calls Factory: conversation.user.text
│     │        ├─ key=command [CommandNodeView]
│     │        │  └─ conversation.chat.commandview
│     │        └─ key=turn-tail [TurnTailNodeView]
│     │           ├─ conversation.chat.turnTail
│     │           └─ conversation.chat.assistant-actions
│     └─ id=trajectory [TrajectoryView / TrajectoryTable]
│        └─ conversation.trajectory.images [MessageImages]
├─ conversation.composer
│  ├─ conversation.approval.detail
│  └─ conversation.plan-review.actions
├─ conversation.composer.bar [InputBar]
│  ├─ conversation.input.attachments
│  ├─ conversation.input.permission
│  ├─ conversation.input.plan
│  ├─ conversation.input.model
│  │  └─ conversation.input.model.label
│  ├─ conversation.input.overlay
│  ├─ conversation.composer.dock
│  ├─ conversation.input.left
│  └─ conversation.input.right
├─ conversation.input.dock
│  └─ id=queue [QueueDock]
│     └─ calls Factory: conversation.user.text
├─ conversation.hero.brand.mark
├─ conversation.hero.workspace
│  └─ conversation.hero.workspace.directoryFlow
├─ conversation.hero.agentPreset
└─ conversation.hero.options

Factory: conversation.chat.user.content [UserContent]
├─ conversation.chat.user.images [MessageImages]
└─ calls Factory: conversation.user.text

Factory: conversation.user.text [UserText]
└─ conversation.user.text.node [keyed, inline]

Factory: conversation.images.content [MessageImagesContent / ImageGallery]
Factory: conversation.chat.reasoning.content [ReasoningContent / MarkdownText]
```

4 つすべての画像スロットは既定で `MessageImages` になり、呼び出し元の画像参照と認可済みローダーを伴って `conversation.images.content` を呼ぶ。ユーザー画像と出力画像の置き換えは独立している。`conversation.user.text` Factory はユーザー・コンテンツ、Queue、Goal で共有され、その `inline: true` のキー付きスロットはブロック・ラッパーではなく span アンカーとインラインのエラー・プレースホルダーをレンダリングする。

生成された[クライアント inspect カタログ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/extensions/cordis-client-runner/src/client/slot-catalog.ts)はすべてのシェル分岐を網羅し、各キーについての完全な契約である。カーディナリティ、スコープ、オーナー props、標準 props、現在の占有者、宣言オーナー、置き換えリスクを含む。実行中の動的パッケージは `cordis_inspect what:"client"` でライブ・ツリーと正確なキーを照会できる。ソース・カタログは `pnpm run gen-client-catalog` によって `SlotMap` 宣言と `slots.register()` 呼び出し箇所から生成される。

## 拡張ルール

- 別のフィーチャー・パッケージをインポートするのは、`import type` による宣言のためだけにする。そのランタイム値をインポートまたは再エクスポートしてはならない。
- 新しい子スロットは、その位置を所有してレンダリングするコンポーネント内でのみ宣言する。他のパッケージは `ctx.slots.inject()` で待機し、`ctx.slots.register()` を通じてコントリビュートする。
- ビジネス状態とトランスポート状態は、所有する Cordis サービスまたは Client モデルに保つ。スロット・ストアは共有の閲覧状態とインタラクション状態だけを保持する。
- observable ソースとスナップショットのアイデンティティを変更間で安定させる。値が変わるたびに同じソースを通じて再公開する。
- UI ドメイン間では JSON 互換のデータとコールバックを渡す。`hooks` 区画は素の observable のための唯一の例外である。React コンテンツはスロットを通じて移動する。
- `single` と占有済みのキー付きセルは置き換えポイントとして扱う。追加型の拡張にはリスト id または未占有のキーを使う。
