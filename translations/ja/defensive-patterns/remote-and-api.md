# Remote と API のルール

日本語 | [English](../../../docs/defensive-patterns/remote-and-api.md) | [中文](../../../docs/defensive-patterns/remote-and-api.zh.md)

> これは非公式の日本語訳です。英語版（[docs/defensive-patterns/remote-and-api.md](../../../docs/defensive-patterns/remote-and-api.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

`packages/api` レイヤーと、それが公開する Remote メソッドに関するルールです。[api-gateway.md](../api-gateway.md) が呼び出しモデル自体を説明します。以下の各節は、要件、レビュアーによる判定方法、対象外のもの、適用される例外とその承認者、正しい例と誤った例を示します。

## 新しい Remote ケイパビリティは独自のサービスを持つ

新しい Remote ビジネス・ケイパビリティは新しいサービス（独自のパッケージまたは独自のオブジェクト）であり、`@Remote` がそのサービスの具象メソッドを印します。既存の大きなサービスを 1 メソッドずつ成長させると、1 つのオブジェクトが auth、Session、workspace、イベント輸送を同時に担うことになり、ドメインごとのアクティベーション・ポリシーと履歴ページネーションが混ざり、後から 1 つのケイパビリティへ加える変更がそれらすべてに触れます。分割は呼び出し元の数ではなくビジネス境界で行います。今日 1 つの呼び出し元しかないケイパビリティも独自のサービスを持ち、所有者が変わるエンドポイントは、離れるオブジェクトに加わるのではなく新しい所有者へ移ります。

- 判定方法: 新しい Remote メソッドをどのオブジェクトが保持するか。すでに別の業務ラインを所有するサービスに新しい業務ラインのメソッドを追加することが欠陥です。修正は、新しいケイパビリティの名前を持つサービスを作り、その具象メソッドに `@Remote` を付けることです。移行も同じように判定します。移動したメソッドは、`packages/subagent/subagent` の `SubagentRuntime` のように新しい所有者に現れ、離れたオブジェクトには現れません。
- 対象外: メソッドがすべて 1 つのビジネス境界に奉仕するサービスは分割せず、そのケイパビリティの新しいプライベート・ヘルパーはそこに属します。独自のプロトコルを必要とするデータも、新しいサービスの Remote メソッドにはなりません。Session イベント・ストリーム、ページネーション、増分 reduce、プロジェクション、エンティティ・サブストリームは、それぞれ別の登録モデルを保ちます（[ゲートウェイの境界](../api-gateway.md#boundaries)）。
- 例外: 記録された例外はありません。呼び出し元が 1 つであることはマージの理由になりません。ビジネス境界自体が争われる場合、ドメイン所有者が変更の着地前にアーキテクチャの問いとして決着させます。
- 修正方法: ケイパビリティ独自のサービス・オブジェクトまたはパッケージを作り、その具象メソッドに `@Remote` を付け、呼び出し元が `ctx.remote.<namespace>` を通じて到達できるようにします。

subagent ケイパビリティは独自のサービスを所有し、すべての Remote メソッドはそのサービスの具象メソッドです:

```ts ignore-check
import { Remote, TypertRemoteService } from '@deepseek-ai/dsh-typert-protocol'

interface SubagentList { ids: string[] }
interface PromptRequest { text: string }
interface PromptResult { accepted: boolean }

class SubagentRuntime extends TypertRemoteService {
  @Remote list(): SubagentList { return { ids: [] } }
  @Remote prompt(request: PromptRequest): PromptResult {
    return { accepted: request.text !== '' }
  }
  @Remote interruptByParent(id: string): void { }
}
```

誤ったバージョンは、すでに他のドメインを運ぶオブジェクトに同じケイパビリティを追加します:

```text
class ApiProxy {
  @Remote goalList(): GoalList            // goal domain
  @Remote subagentPrompt(request): ...    // subagent domain
}
```

1 つのオブジェクトが両方のドメインを担うため、どちらかのケイパビリティのアクティベーション・ポリシー、ページネーション、イベント処理への変更が同じ場所に着地し、どちらのケイパビリティも他方なしには移動できません。

## `@Remote` はケイパビリティのメソッドを印す

`@Remote` はレビュアーが個別にチェックする 5 つの規律を運びます。ケイパビリティを実行する具象メソッドを印すこと。クライアント向けの型が手書きのコピーではなく、生成器を通じて既存の定義から来ること。ペイロードが JSON であり、Remote メソッドが出力側のデコードを行わないこと。既存のメソッド名とフィールド名が追加のみであること。そして、メソッドが触れるすべてのリソースを名指しし、呼び出し元を呼び出しからのみ読むことです。各規律は異なる失敗を取り除きます。ラッパーはどの操作が公開されているかを隠し、2 つ目の手書き型セットは生成されたものから乖離し、出力側のデコードは Gateway がすでに実行する codec の隣に 2 つ目の結果経路を加え、リネームは同じ変更ですべての呼び出し元を壊し、名前のないリソース、フィルタされていない集約、引数で供給された呼び出し元は、非所有者が、どのバインディングも許可していないものに到達できるようにします。

- **ケイパビリティのメソッド。** 注釈は操作を実行する具象メソッド（`list`、`prompt`、`interruptByParent`）に付き、実際の実装へ転送するラッパーには決して付きません。装飾されたメソッドの本体で判定します。委譲するだけの本体は、注釈が誤ったメソッドに付いていることを意味します。
- **推論からの型。** クライアント向けの型は、2 回目に書くのではなく、生成器が生成するものからエイリアスします（`type SessionRemote = TypertClientRemote['session']`）。フィールドが生成された Client 型を繰り返す手書きの宣言で判定します。[typert.md](../subsystems/typert.md) が生成マップとその宣言マージを所有します。
- **ワイヤー上は JSON、出力側のデコードなし。** Remote メソッドの**ワイヤー・ペイロード**は JSON で、バイナリ・コンテンツはフィールド・レベルで表現します。Host のシグネチャは Gateway がすでにサポートするもの（Host オブジェクトに解決されるルックアップ引数、末尾の `AbortSignal`、`@Remote({ mode: 'stream' })` で宣言されたストリーム）を保ち、`@Remote` は出ていく経路でデコードを行いません。生成された codec と Gateway が結果を検証します。デコレーターを伴うメソッドのペイロードにおけるデコード、ストリームの再組み立て、または非 JSON 値そのもので判定します。[プログラミング・モデル](../api-gateway.md#programming-model) が Remote メソッドを越えてよいものを所有します。
- **既存の名前は追加のみ。** 移行は Client がすでに呼び出している Remote メソッドやフィールドをリネームしません。新しいセマンティクスは新しいエントリーを得て、リネームは独自のレビューを持つ独自の変更です。`ctx.remote.<namespace>` を通じてすでに公開されている名前をリネームまたは削除する diff で判定します。
- **呼び出し元の同一性とアクセス。** すべての Remote メソッドはアクセス制御の下にあります。`@Remote({ resources })` / `@RemoteScope(key, { resources })` のリーダーと自動解決は、メソッドが触れるすべてのリソースを名指ししなければなりません。アクセス制御は省略されたリソースを決してチェックしないからです。複数のリソースを返すメソッドは、各項目を `this.ctx.remoteAccess.allows(resource)` でフィルタします。呼び出し元は `this.ctx.invocation.peer.principal` からのみ読み、引数からは決して読みません。ユーザー・メッセージ・ソースを構築する Remote メソッドは、`principalOf(this.ctx.invocation)` を通じてそれを記録し、principal を関数引数として渡すのではなく、構築したソースを下へ渡します。すべての呼び出し経路は Peer を明示的に名指しします。解決されたリソースが触れるものの 1 つを省略するメソッド、フィルタされていない集約、ワイヤー・データやヘルパー・パラメーターを通じて読まれた principal、`principalOf` なしで Remote メソッド内に構築されたユーザー・メッセージ・ソースで判定します。[Peer principal の Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-10-12-peer-principal-and-remote-access.md) が設計を所有します。
- **対象外。** デコレーターのないメソッドはインプロセス呼び出し元を持つ通常のサービス・メソッドのままであり、5 つの規律は何が Remote メソッドになるかを統治し、サービスが何を実装してよいかではありません。
- **例外。** リネームや新しいデコード経路は、その利点と既存の呼び出し元への影響を先に明示し、ドメイン所有者の確認を必要とします。著者が自己承認することは決してありません。

生成されたマップがクライアント向けの型の唯一のソースであり、デコレーターは具象メソッドに付き、ペイロードは JSON のままです:

```ts ignore-check
import { Remote, TypertRemoteService, type TypertClientRemote } from '@deepseek-ai/dsh-typert-protocol'

// Client-facing types come from the generated map, never from a second hand-written set.
type SessionRemote = TypertClientRemote['session']

class SessionController extends TypertRemoteService {
  @Remote('search')
  search(query: string): SessionSearchValue {
    return this.runSearch(query)
  }
}
```

手書きの 2 つ目の型セット、出荷済みの名前のリネーム、出力側のデコードは、それぞれ別の規律を壊します:

```text
type SessionRemote = { list(): ...; rename(id: string): ... }   // hand-written copy
@Remote('renameSession') rename(...)                            // shipped name changed
@Remote decode(payload: Uint8Array): Result                     // output-side decode
```

手書きのコピーはビルドを失敗させずに生成型から乖離し、リネームは既存のすべての Client 呼び出しを一度に壊し、デコードは生成された codec を 2 つ目の未チェックの結果経路に置き換えます。

## `packages/api` は汎用的なオーケストレーションだけを運ぶ

`packages/api` 配下のパッケージ（controllers、`gateway`、`remotes`）は、トランスポート、呼び出しの形、Session ライフサイクル、その他の汎用的なオーケストレーションを運びます。ビジネス分岐がそこで集約されることはありません。具体的なビジネス状態に依存する決定、1 つの機能に束縛された処理、機能ごとのフィルターは、自身のパッケージまたはモジュールに属し、投影されたデータとして、または登録サーフェスを通じてこのレイヤーに届きます。基準はロジックがどこに属するかであり、ファイルがどこにあるかではないため、1 つの機能にしか意味を持たない分岐は、それを保持するファイルが汎用的であっても欠陥です。

- 判定方法: 分岐が条件をどこで読むか。機能状態に対する `if` や `switch`、機能ごとのフィルター、`packages/api` 内の 1 つの機能の名前を持つヘルパーが欠陥です。同じ決定を所有パッケージで行い、投影されたデータまたは登録されたエントリーとしてここで消費する場合はそうではありません。パッケージの棚卸しが影響を示します。`packages/api` は 9 つのパッケージ（`account-controller`、`gateway`、`job-controller`、`remotes`、`session-controller`、`settings-controller`、`terminal-controller`、`workspace-controller`、`workspace-files`）を持ち、`session-controller` が 34 個の `src` ファイルで最大です（terminal-controller 12、gateway 11、workspace-controller 10、workspace-files 8、job-controller 8、remotes 4、settings-controller 3、account-controller 2）。
- 対象外: 汎用的な Session ライフサイクル、トランスポート、ストレージの抽象化は残ります。`gateway` のディスパッチ、`remotes` の選択、共有の session-list とカタログの読み取りはインフラストラクチャであり、ビジネス・ロジックではありません。[ビジネスの所有](../api-gateway.md#business-ownership) が Gateway 側から同じ分割を述べています。
- 例外: 既定ではありません。ここにビジネス分岐を追加するには独立パッケージのレビューが必要で、そのレビューは、なぜそのロジックを機能を所有するパッケージに置けないかに答えます。
- 修正方法: 決定を所有パッケージへ移し、その結果を投影されたデータとして公開するか、このレイヤーがすでに注入しているレジストリに登録し、それを読むオーケストレーションだけを残します。

所有パッケージが決定して自身のユニットを登録し、コントローラーは投影された値だけを読みます:

```ts ignore-check
// packages/workspace/<feature>/src/index.ts: the rule lives with the feature it serves.
export const durablePageProjectionDefinition = {
  key: 'durablePage',
  stateSchema: durablePageState,
  init: () => ({ durable: false }),
  apply: applyDurablePage,
  stateVersion: 1,
}
await ctx.sessionProjections.register(durablePageProjectionDefinition)

// packages/api/session-controller: generic orchestration reads the projected value.
const durablePage = await ctx.sessionProjections.register(durablePageProjectionDefinition)
const { durable } = durablePage.stateOf(session)
```

誤ったバージョンは共有レイヤーの内部でルールを決定します:

```text
// packages/api/session-controller: a feature branch inside the shared layer
if (session.page.transport === 'durable') { /* durable-transport handling */ }
```

ルールは 2 つ目の置き場所を得て、それへの後続のすべての変更は両方を見つけなければならず、共有レイヤーは 1 つの機能のセマンティクスを運び続けます。
