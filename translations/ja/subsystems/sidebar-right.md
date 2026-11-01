# 右サイドバー

日本語 | [English](../../../docs/subsystems/sidebar-right.md) | [中文](../../../docs/subsystems/sidebar-right.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/sidebar-right.md](../../../docs/subsystems/sidebar-right.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

右サイドバーは Web Client の Session ごとのドッキング・サーフェスである。会話の横に並ぶペインとタブの列で、アドレス指定されたコンテンツ — ワークスペース・ファイル、ディレクトリ・ツリー、製品自身のページ — が開き、分割され、浮遊し、閉じる。[`dsh-client-ui-sidebar-right`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-right/README.md) はサーフェス、タブ種別レジストリ、ナビゲーション・サービスを所有し、[`dsh-client-ui-dockkit`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-dockkit/README.md) はその内部レイアウト・エンジンであり、[`dsh-client-resources`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/resources/README.md) はアドレスをあらゆるコンポーネント向けのライブ値に変換し、[`dsh-api-workspace-files`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/workspace-files/README.md) は Host ワークスペース・サービスと Client の `file` リソース・プロバイダーの両方を提供する。

このページはサブシステムの契約のリファレンスである。アドレス、タブ種別の登録、ナビゲーション・サービス、拡張スロットとそのオーナー props、リソース・モデル、Workspace Files サービス、出荷される種別、意図的に作られていないものを扱う。レイアウト・エンジン、フレーム、サーフェスがどのように組み合わされるかは [Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/feature/2026-09-04-right-sidebar-docking-infrastructure.md) にあり、スロットの仕組みは[スロット・リファレンス](slots.md) にある。

## 位置と所有権

ドッキング・サーフェスは Session ごとに 1 つ存在し、session スコープのスロット・ストアに保持され、`rightbar.session` が描画する。root スコープの `rightbar` コントローラーは、選択中の Session と、初期化済みの `keepMounted` ボディを持つバックグラウンド Session のために別々のシートを保持する。表示されるのはフォアグラウンドの Conversation だけである。リロードは保存済みレイアウトを復元する（[状態](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-right/README.md#state)）。サーフェスのすべての変更は、キットの純粋なプランナーが計算する 1 つの履歴エントリーとして記録される。ドッキング済みペインが空のままになることはなく、空の root ペインは登録済みガイド・エントリーから選ばれた既定ページを受け取る。

タブ種別は、定義の `id` を共有する 2 つの登録である。`ctx.sidebarRightTabs` の静的定義が、その `kind` がどのアドレスを開くかを示し、キー付きスロット登録がそのボディを提供する。フレームワークはライブのサイドバー、ペイン、タブ情報のために `useTabInfo()` を注入する。各タブ種別は自身の状態をスロット・ストアに保つ。パッケージは互いの宣言を型としてのみインポートする。

| パッケージ | 役割 |
|---|---|
| [`client/ui-sidebar-right`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-right/README.md) | パネルとレールのシート、レイアウト・ストア、`ctx.sidebarRightTabs`、`ctx.sidebarRight`、Tab ドメイン、ガイド種別 |
| [`client/ui-dockkit`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-dockkit/README.md) | 純粋なレイアウト・エンジンと React サーフェス。`ui-sidebar-right` の内部依存であり、安定したインターフェースではない |
| [`client/resources`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/resources/README.md) | `ctx.resources`、`useResource`、プロトコル → 値の名簿 `ResourceProtocolMap` |
| [`api/workspace-files`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/workspace-files/README.md) | Host の `ctx.workspaceFiles`、`workspaceFiles` Remote 名前空間、Client の `file` リソース・プロバイダー |
| [`util/workspace-path`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/util/workspace-path/README.md) | ファイル・アドレスの文法: `fileAddressFor`、`parseFileAddress` |
| [`client/ui-sidebar-documentpreview`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-documentpreview/README.md), [`client/ui-sidebar-files`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-files/README.md), [`client/ui-sidebar-browser`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-browser/README.md) | 出荷される `text`、`files`、`browser` 種別 |

## アドレス

すべてのタブはアドレス文字列で開かれ、そのアドレスがタブのコンテンツ・アイデンティティである。2 つの系統がある。

**リソース・アドレス**は `dsh-resource://<type>/…` URL である。host がリソース・プロトコル — `ResourceProtocolMap` のキー — を指名し、それ以降はすべてプロトコル自身のパスである。1 つのスキームがすべてのプロトコルにサービスするので、プロトコルを追加すると host が増え、スキームが増えることは決してない。`file` プロトコルのパスはスコープで始まる。`session/<sessionId>` の後にそのセッションのワークスペース・ルートからの相対パスが続く（`dsh-resource://file/session/abc/src/notes.txt`）か、`absolute` の後に先頭の `/` を落とした絶対パスが続く（`dsh-resource://file/absolute/home/ys/notes.txt`、Windows では `dsh-resource://file/absolute/C:/x/y.txt`）。すべての id とパス・セグメントはコンポーネント・エンコードされ、`:` はドライブ文字のためにリテラルのまま保たれる。`fileAddressFor(sessionId, cwd, path)` が 1 つを構築する。相対パスまたはワークスペース内の絶対パスは `session` 相対になり、それ以外の絶対パスは `absolute` になる。`parseFileAddress(address)` はそれを読み戻すか、`undefined` を返す（[文法](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/util/workspace-path/README.md)）。

**ページ・アドレス**は、リソースではなく種別で開かれたタブに対してサイドバーが記録するものである。`sidebar://<kind>` であり、`openTab(kind)` が実行されたときにサイドバー自身が書き込む。呼び出し元が構築することはなく — ガイドとファイル・ツリーは `openTab('guide')` と `openTab('files')` として開かれる — 他のナビゲーション・アドレスは存在しない（[未実装](#not-built)）。

タブのアイデンティティは `(kind, address)` の組である。レジストリの claim はアドレスをそのままレコードの `contentId` として使うので、同じ種別で同じアドレスを開くと既存のタブが見つかり、2 つの種別で同じアドレスを開くと 2 つのタブになる。

## タブ種別の登録

`ctx.sidebarRightTabs.register(definition)` は、呼び出し元のライフタイムの間、ある種別の 1 つの実装を登録してディスポーザーを返す。呼び出し元は自身の `ctx.effect` の内側でそれを保持するので、実装はそれを提供したプラグインと同じ期間だけ存続し、同じ `id` の 2 回目の登録は例外を投げる（[拡張シート](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-right/README.md#extension-seats)）。定義は静的である。ランタイム・フックはなく、タブごとやセッションごとのものもない。

| フィールド | 意味 |
|---|---|
| `id` | 実装のアイデンティティで、すべての登録を通じて一意。パッケージ名が自然な値である（`@deepseek-ai/dsh-client-ui-sidebar-files`）。ボディとタイトルがこのキーの下に登録される。 |
| `kind` | 種別の判別子。そのタブが何であるか、`openTab` が何を指名するか。一意ではない。拡張が組み込みの kind を引き継ぐことができる。出荷される kind は `guide`、`text`、`files`。 |
| `patterns` | 種別が認識する任意のリソース・アドレス・グロブ。種別で開かれるページ種別はこれを省略する。`:` を含むパターンはアドレス全体に一致し（`dsh-resource://file/**`）、含まないパターンは URL のパスに任意の深さで一致する（`*.md`）。URL でないアドレスはそのようなパターンに一致しない。マッチングは大文字小文字を区別せず、ドットファイルを隠さない。構文は picomatch の POSIX 方言である。 |
| `priority` | 3 つのリテラル・バンドのいずれか。`extension`（既定かつ最高。製品外の種別は出荷されるすべてのビューアーより上位）、`builtin`（製品とともに出荷される種別）、`fallback`（より具体的なものが勝つべきプレーン・コンテンツ・ビューアー）。 |
| `canOpen(address)` | グロブ一致に対する任意の同期拒否。すべてのルーティング判断で実行される。 |
| `title(address)` | チップのテキスト。タブが開いたときにレイアウト・レコードにキャプチャされ、書き換えられることはない。 |
| `keepMounted` | 非表示、Session 切り替え、ドッキングをまたいで訪問済みボディを遅延保持する任意の設定。所有する View が Session 参照を保持する。 |
| `guide` | ガイド・ページ用の任意のエントリー・ボックス: `{ order, title(), description?(), icon? }`。ボックスを選ぶと提供元の種別がページとして開く。省略するとページに出ない。 |

アドレス全体のパターンは生のアドレスに一致する。picomatch の globstar（`**`）は、`dot: true`（ドットファイルを含めるが、その除外は変えない）でも単独の `.` と `..` セグメントを除外する。リソース固有のコードがアドレスを解析し、Host がファイル読み取り権限を強制する。

ルーティングは順位付きの claim である。`candidates(address)` は、パターンが一致し `canOpen` が拒否しない種別を、バンド、次に最長一致パターンの長さ、次に登録順で順位付ける。`claim(address, kind?)` は最初の候補、または指名された `kind` をそのまま選び（グロブはスキップされ、`canOpen` は依然として適用される）、`{ kind, contentId: address, title }` を返す。どの種別も claim しないアドレスは例外を投げる。それは配線の誤りであり、ユーザー・エラーではない。

1 つの `kind` は同時に 1 つの `builtin` と 1 つの `extension` 登録を持てる。claim、`get(kind)`、`openTab(kind)`、ガイド・ページでは extension が有効であり、シートは有効な定義の `id` の下でタブのボディとタイトルを見つけるので、スロットの優先度は関与しない。extension が登録解除されると builtin が再開する。kind 上の他のすべての衝突と、すべての重複 `id` は例外を投げる。

```ts ignore-check
import type { Context } from '@deepseek-ai/cordis'
import type {} from '@deepseek-ai/dsh-client-ui-sidebar-right/client'

export const inject = ['sidebarRightTabs', 'slots']

export function apply(ctx: Context): void {
  ctx.effect(() => ctx.sidebarRightTabs.register({
    id: '@acme/dsh-client-ui-image',
    kind: 'image',
    patterns: ['*.png', '*.jpg', '*.gif', '*.svg'],
    canOpen: address => address.startsWith('dsh-resource://file/'),
    title: address => address.slice(address.lastIndexOf('/') + 1),
  }), 'image type')
  ctx.effect(() => ctx.slots.inject('sidebar.right.pane.tab', () => ctx.slots.register(
    { name: 'sidebar.right.pane.tab', key: '@acme/dsh-client-ui-image' },
    ImageBody,
  )), 'image body')
}
```

## ナビゲーション: `ctx.sidebarRight`

2 つのオープンがナビゲーション・コントローラーであり、この列へのあらゆる入口がそのどちらかを呼ぶ。`dsh-resource://` アドレスには `openResource(address, options?)` — 会話のファイル・リンク、ツール行の行参照、ファイル・ツリーの行 — を、ページには `openTab(kind, options?)` — ストリップの追加コントロール、ガイドのエントリー・ボックス、Assistant Markdown 内の HTTP(S) リンク — を使う。どちらも 4 つのステップを 1 つの履歴エントリーとして実行する。claim（レジストリがリソースの種別を順位付けるか、指名された `kind` の有効な実装が答える）、同じ `(kind, address)` を既に表示しているタブへのフォーカス、そうでなければ新しいタブの着席、カラムの展開。その後、ナビゲーションを Tab ドメインに記録する（[サービス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-right/README.md#ctxsidebarright)）。ユーザーが見えないコンテンツは開かれないので、折りたたまれたカラムは同じステップで展開する。ただし、オープンがフローティング・タブを置き換える場合、またはフローティング・パネル内のタブを reveal する場合は除く。既に画面にあるタブの reveal は、そのタブを保持するペインを 1 回光らせる。これはすべてのタブ種別について、その Session が画面にある間だけ行われる。タブを切り替えたりカラムを展開したりする reveal は光らない。`openResource` は `dsh-resource://` の外のアドレスやどの種別も claim しないアドレスで例外を投げ、`openTab` は何も登録していない kind で例外を投げる。どちらも配線の誤りであり、ユーザー・エラーではない。

| オプション | 意味 |
|---|---|
| `paneId` | 新しいタブをこのペインに着地させる。既定はアクティブなドッキング済みペイン（フローティング・ペインがアクティブな間は最初のドッキング済みペイン）。 |
| `replaceTab` | 新しいコンテンツがこのタブの場所を取り、タブは同じステップで閉じる。ドッキング済みタブのペインとストリップ・スロット、またはフローティング・タブのパネルを、ペイン id と矩形を保ったまま引き継ぐ。reveal の場合はタブは開いたままになる（[セマンティクス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-right/README.md#ctxsidebarright)）。 |
| `revealIfOpened` | 既定は `true`。同じ `(kind, address)` を既に表示しているタブにフォーカスし、`params` を渡す。`false` は関係なく別のタブを開く。 |
| `preferNewPane` | 通常のペイン予算と部屋の規則の下で新しいドッキング済みペインを優先する。分割できない場合は対象ペインにフォールバックする。`replaceTab` とともに指定すると無視される。 |
| `destination`（`openResource` のみ） | `'sidebar'` または `'float'`。アクティブなドッキング済みペイン（フローティング・ペインがアクティブな間は最初のドッキング済みペイン）または新しいフローティング・パネルに落ち着き、上記 4 つのオプションを無視する。既にそこにコピーがあれば reveal し、なければもう一方の host のコピーがタブ id とボディ状態とともにそこへ移動し、それもなければ新しいタブが開く。`'float'` は、新しいタブでも移動でもカラムの展開状態を変えない（[セマンティクス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-right/README.md#ctxsidebarright)）。 |
| `kind`（`openResource` のみ） | claim を順位付けする代わりに、開く種別を指名する。有効な実装がアドレスを開き、その `canOpen` は依然として適用される。 |
| `params` | ボディへのナビゲーション・パラメーターで、`navigation.params` として届けられる。`openResource` はマージ拡張可能な `SidebarRightResourceParamsMap` を通じてリソース種別ごとに型付けし（テキスト・プレビューは `{ line?: number }` を宣言する）、`openTab<K>` は `SidebarRightTabParamsMap` を通じて kind ごとに型付けし、何も宣言しない kind では `undefined` である。ボディは両方のユニオンである `SidebarRightNavigationParams` を読む。値は慣習上 JSON 形状であり、実行時には検証されない。 |

配置はタブ種別ではなく呼び出し元に属する。[ファイル・ナビゲーターのリファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-files/README.md#the-navigator)が host 所有のオープン・コールバックを定義し、[ガイドのリファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-right/README.md#the-guide)がガイドの置き換えと明示的なクローズを定義する。

`close(tabId)` はタブを閉じる。`hide(tabId)` はフローティング・タブを閉じるか、そのドッキング済みカラムを折りたたむ。`active()` はアクティブ・ペインのアクティブ・タブを返す。`isExpanded()` と `toggleExpanded()` はカラムを読み取り、反転し、反転はシーケンスに記録される。読み取りは Session がない場合に `undefined` または `false` で答える。書き込みはマウントされた Session サーフェスを必要とし、誰も描画しないサーフェスに書き込むのではなく、ない場合は例外を投げる。

`focus(tabId)` はタブをそのペインのアクティブ・タブにする。`split(paneId?)` はアクティブなドッキング済みペイン、または指名されたペインを分割し、新しいペインの id を返す。ペイン予算またはカラムの幅が分割を禁じる場合は何も記録せず `undefined` を返す。`float(tabId, rect?)` はタブをフローティング・ペインに持ち上げる。`dock(paneId)` はフローティング・ペインをドッキング領域に戻し、カラムを展開する。4 つすべてがストアの既存アクションを実行し、それぞれ 1 つの履歴エントリーを記録する。存在しない、または既に要求された状態にある対象は no-op であり、`open` と同様、マウントされた Session サーフェスがなければ例外を投げる。`TabId`、`PaneId`、`TabRecord`、`FloatRect` はパッケージの `/client` エントリーから再エクスポートされるので、呼び出し元は dockkit をインポートする必要がない。

`focusedTarget(element?)` は、埋め込み iframe を含め、ライブ DOM フォーカス下の可視ページをキャプチャする。サイドバー外または陳腐化したマークアップでは対象を得られない。`commandTarget(element?)` は、サイドバー外で開始されたオープンのために、マウントされた Session のアクティブなドック・ペインも許可する。`SidebarRightTarget` は Session、ペイン、host、タブの出現、ナビゲーション・リビジョンをキャプチャする。`isTargetCurrent(target)` は実行前にそのアイデンティティを確認するので、レコードを開き直したりナビゲートしたりするとキャプチャ済みアクションは無効になる。別のページに再ターゲットすることは決してない。クローズと配置の結果については[コントローラー・メソッド](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-right/src/client/service.ts) を参照。

## スロットとオーナー props

サイドバーは 4 つの拡張スロットを宣言する。そのドキュメント・タブは、以下に示す追加のキー付きドキュメント・ボディを宣言する（[階層](slots.md)）。

| スロット | カーディナリティ | 目的 |
|---|---|---|
| `sidebar.right.pane.tab` | 定義の `id` でキー付け、Session スコープ | 1 つのタブのボディ。シートはタブを、その kind の有効な実装の `id` に dispatch するので、登録者はドッキング済みでもフローティングでも、その kind のすべてのタブを受け取る。実装がボディを登録しなかった kind は、オーナーの「これを表示できるものはない」通知をレンダリングする。 |
| `sidebar.right.pane.tab.title` | 定義の `id` でキー付け、Session スコープ | チップのタイトルで、ボディと同じオーナー共有を持つ。任意。エントリーがなければ、チップは開いたときにキャプチャした `title(address)` のテキストを表示する。ライブ・タイトルを持つ種別はここで自身のストアを読む。 |
| `sidebar.right.tab.guide` | chain、Session スコープ | タブを置き換えずにガイド・タブの内容を置き換える。最初に辞退しないエントリーがボディを取り、そうでなければ出荷されるガイドがレンダリングされる。 |
| `sidebar.right.tab.menu.item` | list、Session スコープ | キット自身のレイアウト・アクションの後に追加されるコンテンツ・レベルのアクション。動作する項目はオーナーの `dismiss()` を呼ばなければならない。 |
| `sidebar.right.tab.document` | ドキュメント実装の `id` でキー付け、Session スコープ | ドキュメント・タブ内で選択されたファイル・レンダラー。親が共有のロードとツールバー制御を所有する。 |

ボディ、タイトル、ガイドの置き換えは、フレームワークが注入する `useTabInfo()` を受け取る。これは `{ sidebar, panel, tab }` を返す。`sidebar` は `expanded` と `fullscreen` を持ち、`panel.id` は格納するペインを指名し、必須の `panel.isFloating: boolean` がフローティング・パネルとドッキング済みペインを区別する。タブが移動すると両方がフックを通じて更新され、`tab.id` は変わらない。`tab` はその他のレコード・フィールドに加えて `visible`、`navigation`、`signal`、`actions` を含む。すべてのタブはフォアグラウンド Session を必要とする。ドッキング済みボディはさらに展開と選択を必要とし、ドッキング済みタイトルは展開を必要とし、フォアグラウンドのフローティングは折りたたみを生き延びる。`signal` はレコードが消えたとき、またはプラグインがアンロードされたときに中断し、非表示や Session 切り替えでは中断しない。`tab.actions` は `openResource`、`openTab`、`close` を提供し、タブ自身の Session に束縛される。オープンは `void` を返し、既定ではその現在のドッキング済みペイン、このタブがフローティングの場合はアクティブなドッキング済みペインを対象とする。`revealIfOpened` は既定で `true`、`replaceTab: true` は新しいコンテンツが同じ履歴エントリーでこのレコードを置き換えることを許す。`destination` を伴う `openResource` は、このタブのペインもこのレコードも渡さない。メニュー・エントリーはプレーンな `tab` と `dismiss` のオーナー・パラメーターを保持する。

`navigation.revision` は、`params` が変わったかどうかに関係なく、そのタブへのナビゲーションごとに増えるので、ボディは「再ナビゲートされた」ということだけで動作できる。アドレスで開かれたタブでは `1`、誰もアドレスで開かなかったレコード — シードされたガイド、またはリロードや undo で復元されたタブ — では `0` である。Tab ドメインは開いているレコードごとに 1 つの出現を保持する。出現したレコードはリソース・モデルでピンされるので、タブを切り替えるとボディはアンマウントされるがコンテンツは失われない。消えたレコードは中断されて破棄される。undo で復元されたレコードは新しい出現である（[Tab ドメイン](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-right/README.md#the-tab-domain)）。

`tab.actions.bindCommands(commands)` は、マウントされたボディのために `SidebarRightTabCommands`（現在は任意の `refresh` コールバック）を提供する。省略された操作は利用できない。返されたディスポーザーは、より新しいボディのコマンドを削除せずにその登録を解放する。タブのライフタイムもそれらを解放する。`tab.refreshShortcut` はページ制御用の有効なリフレッシュ・カタログ・エントリーを提供し、エントリーが利用できない場合は任意である。これらのフィールドは[スロット・オーナー型](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-right/src/client/contract/slots.ts) で宣言される。

## ドキュメント・レンダラー

`text` タブは共有の Document Preview オーナーである。その[ルート登録](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-documentpreview/src/client/index.ts)が `sidebar.right.tab.document` を宣言し、`ctx.documentPreviews` を提供する。レンダラーは自身の effect で `DocumentPreviewDefinition` メタデータを登録し、`ctx.slots.inject('sidebar.right.tab.document', ...)` を通じて待機してから、`key: definition.id` とそのロケール名前空間でコンポーネントを登録する。レンダラーは自身のボディを登録し、子スロットを通じて共有プレゼンテーションを再利用できる。レンダラーを変えてもタブやリソース・アドレスは変わらない。[拡張の決定](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-08-document-preview-operations.md)はプレビュー・ポリシーをリソース所有権から分離している。

[レジストリ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-documentpreview/src/client/document/registry.ts)は一意の `id`、`extensions`、ローカライズされた `title()`、`loading`、任意の `priority`、任意の `wrap` を記録する。大文字小文字を区別しないサフィックス・マッチングは、`extension`（既定）を `builtin` より上位に、次に長いサフィックスを短いものより上位に、次に登録順で順位付ける。タブ kind の置き換えとは異なり、レジストリはすべての実装を利用可能なまま保つ。ツールバーは一致する代替を列挙し、選択をタブごとに記憶する。未知の拡張子はプレーン・テキストを使う。`binaryExtensions` で宣言されたサフィックスは、[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-documentpreview/README.md#what-it-registers) で説明されるように、プレーン・テキストの代替を抑止する。`loading` は `text-pages`、`bytes-complete`、`renderer` のいずれかであり、`wrap` は共有の source-wrap コントロールのサポートを通知する。

[`DocumentPreviewProps`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-documentpreview/src/client/document/contract.ts)は `PropsRuntime<'sidebar.right.tab.document'>` から派生する。オーナーは元の `resourceAddress`、`content`、現在の `wrap`、および `addResource` / `setResources` 依存コールバックを提供する。テキスト・コンテンツは `{ kind: 'text', text, pages: [{ offset, text, lines }], eof }` で、`text` は累積的である。完全なバイトは `{ kind: 'bytes', data }` で、データは `Uint8Array<ArrayBuffer>` である。これらの一時バッファは読み取り専用で借用され、永続レイアウトや Session JSON に入ってはならない。PDF は Worker 転送の前にバイトをコピーし、オーナーのバッファを保つ。子は同じフレームワーク束縛の `useTabInfo` と、グローバルのメタデータ専用 `useResource` を受け取る。親はテキストを `remote.workspaceFiles.read` で、バイトを `remote.workspaceFiles.readBytes` で読み、ページの追加、タブごとのリフレッシュ、ロード状態を所有する。HTML は同じバイト・リーダーに `baseFile` を渡し、Host のコードがパスを解決する。Markdown とコードは追加をまたいで 1 つのインクリメンタル・レンダラーを保持し EOF で確定する。HTML と PDF は完全なバイトを受け取る。

Preview は読み込んだバージョンと、読み取り開始時に観測したバージョンを記録する。自動リフレッシュは既定で有効である。初期メンバー・メタデータがベースラインを確立し、以降の変更がタブを再ロードする。リフレッシュはそのタブだけを読み直し、共有メタデータや別のタブのコンテンツを変えない。読み取りは非トランザクションであり、バージョンは順序付きタイムスタンプではなく不透明な等価トークンである（[リソース観測と Preview RPC](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-08-document-preview-operations.md)）。

ロードを所有するレンダラーは、ファイル・バイトの代わりに `{ kind: 'renderer', revision, loaded, failed, reload }` を受け取る。ボディは注入されたフェイスを呼び出し、リビジョン変更とアンマウントでキャンセルし、表示しているソース・バージョンを `loaded(version)` で報告する。失敗したロードは `failed()` を呼ぶので、後のファイル変更がリトライできる。親は陳腐化した報告を無視し、共有のリロードとソース変更コントロールを保持する。Office はこのモードを使って [Host レンダリングの PDF](office-to-pdf.md) を要求する。そのフェイスは変換済みバイト、フォント・メタデータ、失敗を自身のストアに書き込み、ボディはネストした PDF ビューをレンダリングする。`OfficeFontAction` は `sidebar.right.tab.document.action` を通じて、リロード前にフォント欠落の警告をレンダリングする。[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-documentpreview/README.md#what-it-registers)がロードのライフサイクルを定義する。

## リソース・モデル

モデルは [Client Resources](client-resources.md) で説明されている。この節ではサイドバーが依存する内容を述べる。リソースは 1 つのアドレスであり、リソース・アドレスは `dsh-resource://<type>/…` URL で、その小文字化された host がプロトコル・キーである。プロトコルを所有するクライアント・パッケージは、自身のライフタイムの間 `ctx.resources.register(provider)` で 1 つのプロバイダーを登録する。同じプロトコルの 2 つ目のプロバイダーは例外を投げる（[プロトコルの提供](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/resources/README.md#provide-a-protocol)）。プロバイダーは `{ protocol, open(address, { signal }) }` である。`open` は `RemoteResult` フレーム — 最初に現在の状態、以降の変更ごとに 1 フレーム — を yield し、`signal` が中断すると停止する。失敗は `{ ok: false, error }` フレームであり、例外を投げることはなく、ストリーム内での例外はモデルが捕捉しないプログラミング・エラーである。

`useResource<P>(address)` は、スコープに関係なく、すべてのスロット・コンポーネント上のグローバル標準 prop である。これは `{ status, value, failure }` を返す。アドレスのプロトコルにプロバイダーがない場合、またはアドレスがリソース・アドレスでない場合（`sidebar://guide` はリソースを指名しない）は `none`、最初のフレームまでは `loading`、最新の `ok` 値があれば `live`、最新フレームの失敗が最後の値の隣にあれば `failed` である（[リソースの読み取り](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/resources/README.md#read-a-resource)）。

リソースは保持者 — 購読された `useResource`、または `ctx.resources.pin(address, signal)` — を持つ間開いたままである。最初の保持者がプロバイダーのストリームを開き、後の保持者はそれを共有して最新値をすぐに読み、最後の解放がストリームを中断して値を破棄する。ストリームはコンテンツではなくメタデータを運ぶ。`file` の値は `{ absolutePath, version, bytes? }` であり、コンシューマーは Workspace Files サービスを通じてファイル・テキストをページ単位で自身で読む（[ライフサイクル](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/resources/README.md#lifecycle)）。

## ワークスペース・ファイル

Host の `ctx.workspaceFiles` サービスと生成された `workspaceFiles` Remote 名前空間は、Session ファイルシステム・バックエンドが許可するファイルを読む。`stat(path)` は `{ absolutePath, version, bytes? }` を返す。`read(path, { offset?, limit? })` は行の 1 ページ（`offset` は 1 基準、`limit` は設定されたページ・サイズで上限）を `{ …stat, offset, text, eof }` として返す。`readBytes(path, { range?, baseFile? })` は `maxFileBytes` 未満の完全なファイル、または `maxBytes` 未満のバイト窓を、ネイティブの `Uint8Array` データを持つ `{ …stat, offset, data, eof }` として返す。任意の `range` は 0 基準の `offset` とバイト単位の `length` を使い、`baseFile` は別のファイルのディレクトリーから相対ターゲットを解決する。`list(path)` はワークスペース・ルートの内側に留まり、ディレクトリーの直接の子（`name`、`type: 'file' | 'directory' | 'other'`、`size?`）を、設定された上限まで切り詰めて `truncated` を設定して返す。`changes(path)` は 1 つのターゲットを監視し、Host がその型を判断してディレクトリーをワークスペースに閉じ込める。購読されると `{ kind: 'ready' }` を 1 回 yield し、その後、ペイロードが `{ absolutePath, version }` または `{ absolutePath, absent: true }` である `{ kind: 'change', change }` フレームを yield する（[README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/workspace-files/README.md#use-this-package)）。ファイル読み取りは最終シンボリックリンクを拒否し、転送上限を強制する。`read` はさらに UTF-8 テキストを必要とする。失敗は `workspace-file/*` コードを使う（[失敗](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/workspace-files/README.md)）。

[`dsh-api-workspace-files`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/workspace-files/README.md) は `file` プロバイダーを登録し、`ResourceProtocolMap.file` は `WorkspaceFileStat` を直接指名する。Session アドレスは認可する Session と相対または絶対パスを運び、解決のために変更されずに Host へ渡される。プロバイダーは stat の前に Host の `ready` を待ち、変更を `stat.absolutePath` でフィルターする。裸の `absolute` アドレスは認可する Session を持たず、現在の Session や Tab の Session を借りることなく `workspace-file/unknown-workspace` で失敗する。Global コンポーネントを含むどの UI も、同じ完全なアドレスの観測を共有する。Preview の Remote コールバックはそのアドレス内の Session を使い、`remote.workspaceFiles.readBytes` は Connection のバイナリ RPC トランスポートを通じてネイティブ・バイトを返す。

## 出荷される種別

- **`guide`** — `builtin`、`openTab('guide')` として開かれる。提供された `guide` エントリーごとに 1 つのカプセルの上に、ミュートされたコンパスが `order` 順で並ぶ。短いリストは登録済みの説明を表示し、アイコンがない場合は出荷されるプレースホルダーを使う。カプセルを選ぶと、提供元の種別がガイド・タブの場所にページとして開く。1 つのペインは最大 1 つのガイド・タブを保持し、ストリップの追加コントロールはそのペインにガイドがない間だけ現れる。新しいペインは登録済みの既定ページを受け取る。ガイド・エントリーが 1 つだけならそれを直接、エントリー数が 1 でなければガイドを受け取る（[ガイド](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-right/README.md#the-guide)）。
- **`text`** — `fallback`、`dsh-resource://file/**`、Session アドレスだけを claim する。Document Preview は `useResource<'file'>` を通じてメタデータを観測し、Remote コールバックを通じてコンテンツをロードし、レンダラーの選択、ツールバー、タブごとのリフレッシュ、スクロール、ソース・ナビゲーションを所有する。未知の拡張子はプレーン・テキストとしてレンダリングされる。ドッキング済みとフローティングのペインで、files プラグインの `sidebar.files.navigator` と `sidebar.files.navigator.toggle` Factory を直接レンダリングし、未登録の Factory は何もレンダリングしない（[README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-documentpreview/README.md)）。
- **`files`** — `builtin`、`openTab('files')` として開かれる。ファイル・ページとナビゲーターはペインごとに 1 つのワークスペース・ツリーを共有する。[パッケージ・リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-files/README.md)が遅延リスト、監視、オープン・コールバック、ナビゲーション駆動の可視性を所有する。
- **`browser`** — マルチインスタンスの `builtin`、`openTab('browser', { params: { url? } })` として開かれる。Chat の[リンク設定](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-chat/README.md)が、Assistant Markdown の HTTP(S) リンクをここで開くかどうかを制御する。既定の sandbox 下でパブリックおよびループバックの HTTP(S) ターゲットを受け入れ、ローカル・ファイルは Document Preview に譲って拒否し、アプリケーションが把握する iframe 履歴を使う（[README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-browser/README.md)）。
- **`subagentchat`** — `builtin`、`dsh-resource://subagentchat/session/<child>?parent=<parent>&mode=<mode>`。リソース・プロバイダーが保持し、共有 Conversation Factory を通じてレンダリングされる、明示的にアドレス指定されたサブエージェント Conversation（[README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-subagent/README.md)）。

<a id="not-built"></a>
## 未実装

- クロスデバイスのレイアウト同期。レイアウトはブラウザー・ローカルで、Session ごとに別々の保存状態を持つ（[状態](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-sidebar-right/README.md#state)）。
- `ctx.sidebarRight` 上の読み取り専用レイアウト・スナップショットまたは購読。サービスは操作だけを公開し、dockkit の `LayoutState`/`LayoutOp` は内部である。
- サービス上のケイパビリティ発見配列（`features`）。
- タブ種別の `option` 優先度バンド。何かを claim させずにタブ種別を列挙するものはない。
- レコードの再タイトル化。`title(address)` は 1 回キャプチャされ、ライブのチップはレコードからではなくタイトル・スロットから来る。
- オープン時のタブ実装の指名。`openResource` はたかだか kind を指名し、ドキュメント・レンダラーの選択はファイル・タブのツールバーに属する。
- サービス上のアドレス検索（`find`）。呼び出し元は `revealIfOpened` で開き、サーフェスに重複排除を任せる。
- サイドバー自身の `sidebar://<kind>` の記帳を超えるナビゲーション・アドレス。その文法はナビゲーション・コントローラー全体を待つ。
- ユーザー向けの undo と一般的なクロス種別コンテンツ・ナビゲーション・スタック（[延期](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/feature/2026-09-04-right-sidebar-docking-infrastructure.md#deferred)）。Browser は自身のページ履歴だけを所有する。
