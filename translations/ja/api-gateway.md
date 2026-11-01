# API ゲートウェイ

日本語 | [English](../../docs/api-gateway.md) | [中文](../../docs/api-gateway.zh.md)

> これは非公式の日本語訳です。英語版（[docs/api-gateway.md](../../docs/api-gateway.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

これは Typert API ゲートウェイの現在の状態のリファレンスです。ビジネス・サービスが単項とストリームの Remote メソッドをどう宣言し、ビルドが Host と Client の契約をどう生成し、呼び出しが Connection RPC と `/api` ルートをどう再利用するかを説明します。Session イベント、増分データ、その他のストリーミング・プロトコルはこのドキュメントのスコープ外です。同じ Connection を使ってもかまいませんが、Remote メソッドの記述子は使いません。

## プログラミング・モデル

ビジネス・サービスは `@Remote` または `@RemoteScope` を使って、Client に公開するメソッドを選びます。印のないメソッドは、生成される Client の型や実行時のコントリビューションに入らず、`ctx.remote` から呼び出すことはできません。

`@Remote` は、ルートの Host Context に登録された Cordis サービスへの呼び出しを表します。複雑な Host オブジェクトはワイヤーを直接越えられません。ビジネス・パッケージは `TypertLookupMap` を通じてワイヤー識別子との関連を宣言し、実行時に `ctx.typert.lookups` へ既定の解決プロバイダーを登録しなければなりません。たとえば、Host のシグネチャで `agent` という名前の `Agent` パラメーターは `agentId` ワイヤー・フィールドを生成し、ゲートウェイはビジネス・メソッドを呼び出す前にその id を Host オブジェクトに解決します。Host のコンポジションは `ctx.typert.lookups.configure()` を使って、ビジネス・パッケージが所有するパラメーター名、ワイヤー・フィールド、正準の型シンボルを変えずに、ルックアップ・キーの解決ポリシーを上書きできます。

`@RemoteScope(key)` は、まず `ctx.typert.contexts` を通じて識別子をスコープ付き Context に解決し、その Context からサービスを取得してメソッドを呼び出します。メソッド自体がスコープ付きのコンポジションに依存し、`Agent` のようなオブジェクトを明示的に受け取る必要がない場合に適用されます。

`@Remote` はエクスポート名か `{ name?, mode?, resources? }` を受け取り、`@RemoteScope(key, option?)` はエクスポート名（下の `@RemoteScope('agent', 'current')` のように）か、2 番目の引数として `{ name?, resources? }` を受け取ります。`resources` は、アクセス制御のために 1 回の呼び出しが触れるリソースの識別子を返す、種類ごとのリーダーを保持します。各リーダーは、装飾されたメソッドから推論された型で、そのメソッドのワイヤー引数を自身のパラメーターとして受け取ります。ほとんどのメソッドはリーダーを宣言しません。ゲートウェイは、ルックアップ・パラメーター、`resource` を宣言するスコープ付き Context、`typert.resources` の既定のフィールドやリゾルバーからも識別子を読むためです。その順序は [Typert サブシステム](subsystems/typert.md)が所有します。

サービスは通常 `TypertRemoteService` を継承し、コンストラクターが Cordis のサービス・キーと既定の Remote 名前空間を明示的に束縛します。すでに別の基底クラスを持つサービスは、代わりに `readonly typertRemote = bindTypertRemote(this, serviceKey)` を宣言できます。どちらの形式も検査可能な公開バインディングを残し、コンパイラーがコンストラクターにシンボルを注入することに依存しません。

```ts
import type { Agent } from '@deepseek-ai/dsh-agent'
import { TypertRemoteService, Remote, RemoteScope } from '@deepseek-ai/dsh-typert-protocol'
import type { Context } from '@deepseek-ai/cordis'

export interface CreateGoalRequest {
  objective: string
}

export interface CreateGoalResult {
  accepted: boolean
}

export class GoalService extends TypertRemoteService {
  constructor(ctx: Context) {
    super(ctx, 'goals')
  }

  @Remote('create')
  createForClient(
    agent: Agent,
    request: CreateGoalRequest,
    signal: AbortSignal,
  ): CreateGoalResult {
    signal.throwIfAborted()
    return this.create(agent, request)
  }

  @RemoteScope('agent', 'current')
  currentForClient(): CreateGoalResult {
    return { accepted: true }
  }

  private create(_agent: Agent, request: CreateGoalRequest): CreateGoalResult {
    return { accepted: request.objective.length > 0 }
  }
}
```

Remote メソッドは、同期的に値を返しても、Promise を返してもかまいません。協調的なキャンセルのため、Host のシグネチャの最後のパラメーターは、グローバルな型を使う `signal: AbortSignal` でなければなりません。これは `args` に入らずに記述子に記録され、生成される Client メソッドは省略可能な最後の `AbortSignal` を受け取ります。

`@Remote({ mode: 'stream' })` は、`Iterable`、`AsyncIterable`、`RemoteStream<Out, In>` を返すメソッドを印付けします。ゲートウェイは、Host メソッドが生成した各値（yield された各項目）を、多重化された `/api/remote.mux` WebSocket またはインプロセスのキャリアで届け、生成される Client メソッドは、項目を反復し、同じ論理ストリームの Client から Host へのアップリンク用に `send`、`end`、`dispose` を公開する `RemoteStreamHandle<Out, In>` を返します。`RemoteStream<Out, In>` の 2 番目の型引数はアップリンク項目の型を宣言します。Client が送る各項目はブラウザから届くため、ゲートウェイは Host メソッドが `this.ctx.invocation.uplink<In>()` で読む前に、生成された `In` の codec で各項目を検証します。アップリンクは `args` にもパラメーター・リストにも入りません。フレーム、ハーフクローズ、キャンセル、受信トレイの上限の契約は [ゲートウェイの README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/gateway/README.md) が所有します。

Client は JavaScript の Proxy ではなく、通常のオブジェクト上の具象関数を使います。直接の呼び出しとスコープ付きの呼び出しは、`ctx.remote.<namespace>` と `agentCtx.remote.<namespace>` に現れます。各名前空間は `remote.<namespace>` として登録された、トレース可能な Cordis の子 Service です。Client のアセンブリは `ctx.remote.$mount()` を通じてコントリビューションをマウントし、名前空間は最後のメソッドが撤回された後にアンロードされます。依存関係の宣言は実際の呼び出し元に属します。`ctx.remote.<namespace>` や `agentCtx.remote.<namespace>` を読むビジネス・パッケージだけが、自身の `inject` で `remote` と `remote.<namespace>` の両方を宣言します。コントリビューションをマウントするだけのアセンブリや、その名前空間を呼ばない高レベルのランタイムは、ビジネス・パッケージの代わりに名前空間の依存関係を宣言しません。`@Remote` メソッドがちょうど 1 つのルックアップ・パラメーターを持ち、同名の `TypertContextMap` が同じワイヤー識別子を使う場合、生成されるスコープ付きシグネチャはその識別子パラメーターを省略します。`@RemoteScope` はスコープ付きの呼び出しインターフェースだけを生成します。

```ts ignore-check
import type { SessionId } from '@deepseek-ai/dsh-session/types'
import type { AgentContext } from '@deepseek-ai/dsh-api-session-controller/client'
import type { Context } from '@deepseek-ai/cordis'
import type {} from '@deepseek-ai/dsh-api-remotes/client'

export const inject = ['remote', 'remote.goals']

declare const ctx: Context
declare const agentCtx: AgentContext
declare const agentId: SessionId

await ctx.remote.goals.create(agentId, { objective: 'ship it' })
await agentCtx.remote.goals.create({ objective: 'ship it' })
```

Client アプリケーションは `@deepseek-ai/dsh-api-remotes` だけをアセンブルします。そのパッケージは選択されたビジネス・パッケージの `/remote` サブパスを実行時の値として import し、`ctx.remote.$mount()` を通じてコントリビューションをマウントし、同じファイルからの宣言マージを再エクスポートします。Host の Remote パッケージを追加することは、Client のコンポジションの所有者による明示的な選択です。ビジネス・コンポーネントは Typert ゲートウェイやビジネス・パッケージの Remote JS を別々にロードする必要はありません。

`api-remotes` のアセンブリと `ctx.remote` の契約は React に依存しません。任意の Client アセンブリに見える Host メソッドは、生成時に選択された Remote メソッドに限られます。

## コンポーネントの責任

| 場所 | パッケージまたはエントリー | 責任 |
|---|---|---|
| 共有 | `@deepseek-ai/dsh-typert-protocol` | デコレーター、ゲートウェイのバインディング、マージで拡張できるプロトコル・マップ、呼び出し記述子、プロバイダー型を宣言する。TypeScript の解析を開始せず、Cordis サービスも登録しない |
| ビルド | `@deepseek-ai/dsh-typert-generator` | Host の `ts.Program` から、Remote のシグネチャ、型グラフ、ルックアップ、Context、ソース位置を厳密に解析し、Host と Host-for-Client の成果物を生成する |
| Host | `@deepseek-ai/dsh-typert-registry` と Loader | 生成された Host の記述子、スキーマ、ビジネス・パッケージの登録を `ctx.typert` に置き、ルックアップと Context のプロバイダーを保持する |
| Host | `@deepseek-ai/dsh-api-session-controller` | アプリケーションの Agent／Session 識別子ポリシーを所有し、対応する Typert のルックアップを設定する |
| Host | `@deepseek-ai/dsh-api-gateway` | `ctx.typertGateway` を提供し、Remote エンドポイントを引き受け、リクエスト値を検証し、オブジェクトや Context を解決し、生きた Cordis サービスを呼び出す |
| Host | `@deepseek-ai/dsh-api-remote-access` | ビジネスが登録したロールベースのポリシーを通じて、非オーナーの Peer が呼べる Remote メソッドを決め、リストの結果を絞り込み、各 Peer のアクセス・スナップショットをストリームする。ゲートウェイはこれを必要とする |
| Client | `@deepseek-ai/dsh-api-gateway/client` | `ctx.remote` と `remote.<namespace>` の子 Service を提供し、生成された記述子を具象メソッドとしてマウントし、Connection を通じて呼び出しを開始し、キャンセルする |
| Client | `@deepseek-ai/dsh-api-remotes/client` | アプリケーションが許可する `/remote` のコントリビューションを明示的に選んでマウントし、対応する宣言マージをビジネス・コードに取り込む |
| 両方 | `@deepseek-ai/dsh-client-connection` | RPC キャリア、リクエストの相関、信頼境界、キャンセル、レスポンス・エンベロープ、`/api` の HTTP ブリッジを提供する |

API ゲートウェイ・パッケージは Host のディスパッチャーと Client の Remote エンドポイントをピアのエントリーとして所有しますが、2 つのビルドが同じ `ts.Program` に入ることはありません。Host のエントリーは Client の Cordis `Context` マージを import せず、Client のエントリーは Host のゲートウェイ・サービスを import しません。

## 厳密な生成パイプライン

ルートのビルドは、まず `scripts/generate-typert.ts` を通じてソースのみの Host 解析を実行し、その後 Host と Client のフェイスをコンパイルしてバンドルし、Web をビルドします（[ステップの順序](development.md#typescript-project-layout)）。生成器は Host アグリゲートを唯一の `ts.Program` のシードとして使い、パッケージのコンパイル前に Remote 宣言を書き出します。Host の tsdown パスはメタデータを再度生成せずにデコレーターを下げ、Client のパスはそれらの宣言と実行時のコントリビューションを消費します。

両方の tsdown パスは `vendor/*`、`packages/*/*`、`apps/cli` に一致し、Host のパスは `apps/desktop-host` にも一致します（[ビルドの順序](development.md#typescript-project-layout)）。どちらも、対応する tsc フェイズが `lib/types` に出力した JavaScript だけをバンドルします。ルートの設定は Client の成果物を走査せず、パッケージ名を分類せず、維持されたフィルターを tsdown に渡しません。パッケージ・ローカルの設定は `DSH_BUILD_FACE` に基づいて現在のフェイズのエントリーを返します。通常の Client プラグインは、Client フェイズで Node ローダーのエントリーとブラウザ・バンドルの両方を生成します。

`api/remotes`、`api/gateway`、`api/session-controller`、`api/workspace-controller`（加えて `client/connection`）は TypeScript のフェイスを分割します。`api/remotes` の Client プロジェクトは、パッケージのコンパイル前にビジネス・パッケージ向けに生成された `/remote` 宣言に依存します。ルートのアグリゲートと直接の利用者は、分割パッケージそれぞれの `tsconfig.host.json` または `tsconfig.client.json` を参照しなければなりません。`api-remotes` の `clientBundle(..., { hostPhase: true })` は Host の tsdown 中に Host のエントリーを生成し、Client の tsdown にはブラウザのエントリーだけを残します。Agent／Session のルックアップ・ポリシーは `api-remotes` ではなく `@deepseek-ai/dsh-api-session-controller` にあります。

コントリビューションする各ビジネス・パッケージは、生成ファイルをソース・ディレクトリではなく自身の `lib/` ディレクトリに書き出します:

| ファイル | 利用者 | 内容 |
|---|---|---|
| `typert.host.js` | Host Loader | Host フェイスの実行時リフレクション、厳密な呼び出し記述子、スキーマ登録の値 |
| `typert.host.d.ts` | Host の型システム | Host フェイス向けに生成された宣言 |
| `typert.remote-client.js` | `api-remotes` | 厳密な記述子と実行時の codec を含む、マウント可能な `TypertRemoteContribution` |
| `typert.remote-client.d.ts` | Client の型システム | `TypertRemoteNamespaceMap` と `TypertRemoteScopeMap` の宣言マージ、および Client で安全な型参照 |
| `typert.remote-client.d.ts.map` | エディター | 生成されたメソッド・プロパティを、Host パッケージの Remote メソッド宣言に逆写像する |

ビジネス・パッケージは、Host Loader のエントリーを `./typert` で、Host-for-Client のエントリーを `./remote` で公開します。生成器はこれらのパッケージのエクスポートと公開ファイルの一覧も検証し、対応するエントリーを提供する明示的なコントリビューション・パッケージにだけ成果物を生成します。

Remote Client 宣言のパラメーター名はワイヤー・フィールドに由来し、パラメーターと戻り値の型は元のビジネス・パッケージがエクスポートする Client で安全な型を参照します。宣言マップは、`ctx.remote.goals.create` の背後の生成プロパティを、`@Remote` で印付けられた Host のソース・メソッドへ逆解決します。そのため、宣言マップをサポートするエディターは、生成された `.d.ts` で止まらずに、Client の呼び出しから実際の実装へ移動できます。

厳密な解析は、Remote が具象の実装を持つ公開の非 static インスタンス・メソッドであることを要求します。メソッドはジェネリックにはできず、パラメーターは必須で、単純な識別子の名前を持ち、分割代入、既定値、残余パラメーター、省略可能なパラメーターを使ってはいけません。Typert は通常の JSON 表現可能な型に対して厳密なスキーマを生成します。ワークスペースのクラスのような複雑なオブジェクトは、一意の `TypertLookupMap` 宣言を持たなければなりません。ルックアップと Context のパッケージは、静的な宣言マージと実行時のプロバイダー登録の両方に責任を持ちます。どちらかが欠けると、ビルドが失敗するか、プロバイダーを必要とする最初の呼び出しが失敗します。

## ランタイム呼び出し

Remote 呼び出しは Connection の `/api` ルートを使います。Client の Remote は `connection.rpc.call('/api', '<namespace>/<method>', { args }, signal)` を呼び、HTTP キャリアはこれを `POST /api/<namespace>/<method>` に写像します。ペイロードは名前付きの `args` オブジェクトだけを含みます。

Connection は HTTP ブリッジの前に `/api` の統一的な信頼チェックを実行し、共有の FetchHandler の内側でディスパッチします。Typert ゲートウェイは、厳密な記述子かアクティブな SRC マーカーを持つ 2 セグメントのエンドポイントだけを要求します。バイナリの Remote フィールドを JSON 互換のメタデータと結果相対のバイト添付に投影し、Connection がそれをマルチパートのレスポンスとしてフレーム化します。機能が所有する正確な Fetch ルートは RPC エンベロープの外のレスポンスを扱い、その他のリクエストは 404 を返します。Connection はトランスポート、RPC id、レスポンス・エンベロープ、リクエストのキャンセルを所有し、ゲートウェイは Remote のデータ・プロトコルとビジネスのディスパッチを所有します。Connection のキャリアを置き換えても、Remote の記述子や Client のプログラミング・インターフェースを変える必要はありません。

すべての呼び出しで、ゲートウェイはビジネス・オブジェクトをキャッシュせずに、現在のレジストリから記述子と生きたサービスを解決します。すべての呼び出しは、それが代弁する Peer を名指しし、dispose された Peer は `gateway/peer-disposed` で拒否されます。ゲートウェイは `args` のフィールドが記述子と正確に一致することを要求します。Peer のプリンシパルがオーナーでない呼び出しでは、続いてその呼び出しが触れるリソースを解決し、`ctx.remoteAccess.decide()` に問い合わせます。そのため、拒否された呼び出しはルックアップを解決せず、Agent もアクティブ化しません。ゲートウェイは codec でワイヤー値を検証し、登録済みのルックアップや Context のプロバイダーを通じてオブジェクトやレシーバーを解決し、バインディングが指すサービスのメソッドを呼び出します。Client のゲートウェイは、デコードされたバイナリの結果を生成された結果 codec で検証します。通常の JSON の結果は既存の扱いを保ちます。プロバイダーの欠落、未知の識別子、バインディングの不一致、引数の欠落や過剰、スキーマの失敗、メソッドの欠落は、ビジネス・コードに入る前か、出た後に失敗します。

ルックアップ・プロバイダーの `register()` は、安定した宣言と既定のリゾルバーの両方を提供します。`configure()` は Host のコンポジションが所有するリゾルバーを提供し、非同期に実行されることがあり、エフェクトのライフタイムにスコープされます。設定はプロバイダーのマウントに先行してかまいません。プロバイダーがない場合、呼び出しは引き続き `gateway/lookup-unavailable` で失敗し、設定をアンロードするとプロバイダーの既定のポリシーが復元されます。Session Controller は `agent` と `session` の標準のリゾルバーのセマンティクスを所有します。生きた Agent を再利用し、通常のコールド・セッションを自動的に再開し、同時の再開を重複排除し、サブ agent のルーティングが所有する識別子を拒否します。`session` のルックアップはその Agent の Session を返します。再開の失敗と所有関係のフェンスは、それぞれ独自のコード `session/not-found` または `session/agent-busy` を持つ `RemoteError` を送出し、ゲートウェイはそれをそのままワイヤーにエンコードします。分類されていない throw だけが `gateway/internal` に畳まれます。

Client のコントリビューションをアンロードすると、その記述子と具象メソッドが一緒に削除され、進行中の呼び出しが中止され、外部のコードが保持する古いメソッド・ハンドルは以降の呼び出しを拒否するようになります。Host で撤回された厳密なエンドポイントも SRC の推論に劣化しないため、ホット・アンロードが検証を静かに弱めることはありません。

## SRC 開発フォールバック

Host が `node --import tsx/esm` を通じてソースから起動する場合、Typert のコンパイラー・プラグインは実行されません。標準のデコレーター初期化子は、Service プロトタイプ上のバージョン付き記述子にメソッド名と呼び出しモードを記録し、`TypertRemoteService` または `bindTypertRemote()` が明示的なサービス・バインディングを提供します。そのためゲートウェイは、`ts.Program` を開始せずに、より弱い一時的な記述子を構築できます。記述子の安定した文字列プロパティ名により、`remoteMethods()` は別のインストール済みプロトコル・パッケージのコピーが書いたマーカーを読めます。

SRC フォールバックは、生きた関数から単純なパラメーター名を解析します。パラメーター名が登録済みルックアップの `parameter`（たとえば `agent` や `session`）に一致する場合、そのルックアップの `agentId` または `sessionId` ワイヤー・フィールドを使い、Host 上でオブジェクトを解決します。その他のパラメーターは、循環のない、特殊なプロトタイプを持たない JSON で安全なデータであることだけを検査します。`@RemoteScope` は登録済み Host Context プロバイダーのワイヤー・フィールドを直接使います。SRC は TypeScript の型を読まず、Zod スキーマを生成せず、省略可能なパラメーターを推論せず、分割代入、既定値、残余パラメーター、重複したパラメーター名をサポートしません。

SRC は、ソースから実行される Host プロセスのディスパッチだけを解決します。Client は実行中の Host からデコレーターを発見せず、Client の Remote は厳密な codec を持たない SRC 記述子のマウントを拒否します。その型、codec、Remote 登録の値は、常に最も新しく生成された `lib/typert.remote-client.*` の成果物から来ます。

## 開発モード

Web の開発は 1 つのコマンドで実行し、現在の Host、Client、Web の成果物をビルドし、ソースの Host を起動し、Client のバンドルを再ビルドし続けます:

```sh
pnpm run dev:web
```

`dsh` は tsx を通じて Host のソースを起動するため、Host は SRC フォールバックを使えます。`dev:web` のウォッチャーは、ソースの編集時に client フェイスの型出力、Client プラグインとライブラリのバンドル、Web シェルを再ビルドします。Host のデコレーターを解析したり、Remote Client の DTS を生成したりはしません。それを担うのは、`dev:web` が最初に実行する完全なビルド（または `pnpm run build`）だけです。

契約を変えずに Remote メソッドの実装本体だけを変える場合、Typert ファイルの再生成は不要です。デコレーターの追加や削除、エクスポート名、名前空間、パラメーター、戻り値、ルックアップ、Context、キャンセルのシグネチャの変更後は、順序付けられた lib ビルドを再実行し、Client が新しいコントリビューションをコンパイルしてバンドルする前に Host が厳密な契約を生成するようにします:

```sh
pnpm run build:lib
```

実行中の Client のウォッチャーは、再バンドルするときにこれらの生成ファイルを消費します。`pnpm run build:lib:host` がすでに Host の契約を更新していれば、`pnpm run build:lib:client` が Client 側を完了できます。クリーンな worktree で Host フェイズを省略することはできません。フロントエンドのソースだけを再コンパイルしても、Host のデコレーターから新しい型を推論することはできません。`pnpm run typecheck` は Client の tsc の前にバンドルなしの Typert パスで Host の契約を生成し、CI のビルド・レーンとリリース・ビルドも、パッケージのコンパイル前に同じソースのみの生成を使います。

## 境界

Remote は、1 つのリクエストと 1 つの結果を持つ単項のメソッド呼び出しと、項目が Host → Client に流れ、Client の項目が `this.ctx.invocation.uplink()` を通じて実行中の Host メソッドに届くストリーム・メソッドを扱います。Session のイベント・ストリーム、ページネーション、増分の reduce、プロジェクション、エンティティのサブストリームは、引き続き別のデータ・プロトコルと登録モデルを必要とします。Connection を再利用する場合でも、Remote メソッドを装ったり、呼び出し記述子に入ったりしてはいけません。

API の各層は `remotes → gateway → connection → webserver` として構成されます。BFF と Typert RPC の層は `packages/api` にあり、Connection と WebServer は `packages/client/connection` と `packages/host/webserver` にあります。ネストしたフィールドや配列の要素も含め、`Uint8Array` を含む単項の結果は、[バイナリの Remote 転送](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-17-workspace-file-binary-transfer.md)を使います。RPC エンベロープの外のレスポンスを必要とする機能は、Connection の正確な Fetch ルートを登録します。

ルックアップ・ポリシーはキーごとに設定されるため、すべての `agent` や `session` パラメーターがコールド再開の挙動を共有します。生きたオブジェクトだけを受け入れるには、パラメーターごとまたはエンドポイントごとの明示的なポリシーが必要ですが、それは存在しません。ビジネス・メソッドは、オブジェクトが復元から来たかどうかを推測してはいけません。

### ビジネスの所有関係

`packages/api` はトランスポート、呼び出しの形、セッション・ライフサイクルを所有します。ビジネスのルールは独立したビジネス・パッケージに属し、登録サーフェスや投影されたデータを通じてこの層に届きます。ここの変更が満たさなければならない要件、すなわち 4 つの `@Remote` の規律、追加のみの命名規則、この層が保つ境界は、[Remote と API のルール](defensive-patterns/remote-and-api.md)が所有し、レビューはそこでそれらを検査します。
