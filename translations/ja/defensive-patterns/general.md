# 防御的パターン

日本語 | [English](../../../docs/defensive-patterns/general.md) | [中文](../../../docs/defensive-patterns/general.zh.md)

> これは非公式の日本語訳です。英語版（[docs/defensive-patterns/general.md](../../../docs/defensive-patterns/general.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

苦労して得たバグ分類のルールです。以下の各パターンは、ここで実際に出荷されたか、もう少しで出荷されるところだった欠陥の分類であり、その再発を防ぐルールとして述べています。ライフサイクル、並行性、サブプロセス、teardown のコードを書く前にこれを読んでください。テスト階層の対応物（実際のエントリー経路、世界の検証、リソースの所有）は [testing.md](../testing.md) にあり、これらのルールが参照する型レベルの定義は [subsystems/](../subsystems/index.md) にあります。

## 直交する結果は独立に報告する

1 つの結果は同時に複数のことを示せます。プロセスはシグナルを捕捉したためにタイムアウトし、かつ exit 0 することがあります。各独立した事実（`timedOut`、`signal`、`exitCode`）をそれぞれ表面化し、あるフラグの報告を別の分岐の内側に入れ子にしないでください。呼び出し元が途中で切り詰められた実行をきれいな成功として読むからです。

レビュアーは停止の仕組みではなく報告の形を確認します。各事実が独立に真になり得るか、それぞれが別の条件の分岐に入らずに読めるかです。境界は報告であり、プロセスにどうシグナルを送るか、どれだけ時間を与えるかは別の決定です。

## 公開契約を両側で守る

実装が 1 つの結果の複数の表現を受け取る場合、公開 API を通じて返す前にそれらを正規化します。`LlmAdapter.stream()` の実装は throw するか `finish {kind:'error'|'aborted'}` を発行できますが、`LlmRuntime.stream()` はモデル・リクエストの失敗を終端の finish チャンクとしてのみ公開します。ミドルウェアとコンシューマーの欠陥は引き続き throw されます。これにより、捕捉された例外がプロバイダー、ラッパー、チャンク・ログ、または自身の組み立てから来たのかをコンシューマーが推測する必要がなくなります。正規化された契約は型が定義されている場所に文書化し、実際のコンシューマーを通じてすべてのソース形式を試します。

判定条件は、コンシューマーが公開 API だけから失敗のソースを名指しできるかです。内部ヘルパーは自由に throw してかまいません。要件が拘束するのはコンシューマーが越える境界であり、その背後のすべての関数ではありません。

## 非同期状態は同期状態ではない

`agent.followup()` にはメッセージ単位の完了や結果がありません。バックグラウンド・ジョブの完了は turn の境界と競合し、`reader.close()` は EOF と破棄の両方で発火します。`agent/status` や `whenIdle()` を 1 つの follow-up の結果として扱わないでください。複数のキュー済み follow-up、steering、注入された処理が 1 つの `running` 区間を共有することがあり、キャンセルや破棄は未開始の項目を捨てることがあります。実行を真に所有する自動化の呼び出し元は、その区間を明示的に定義し（たとえば、メッセージの永続的な inbox 受領から次の whole-agent の `idle` まで）、選択した出力をそのメッセージに因果的に帰属させるのではなく、区間全体のものとして記述しなければなりません。このガードは逆方向にも働きます。待機する遷移が決して起こり得ない場合、待機はハングするため、「待つものが何もない」分岐を明示的に処理します。

判定条件は、agent が API の定義しないユニット単位の完了を観測できるかです。区間が真に呼び出し元に属する場合、区間と出力スコープを明示するのは呼び出し元であり、共有の `running` フラグがその明示になることは決してありません。

## タイミングは正しさを支えない

正しさは明示的な呼び出しポイントと明示的な状態から来るものであり、コールバックがイベントループのどこに着地するかからは決して来ません。処理が正しいかどうかを決める `microtask`、`setTimeout`、`queueMicrotask` はプログラムの残りと競合します。軽い負荷では成立し、忙しいときやバッチ化が変わると黙って失敗します。React のコードも同じルールに従います。エフェクトが、設計が明示すべき「レンダー直後」の順序を提供することは決してありません。

判定条件は、正しさが名前付きの呼び出しポイントや状態機械ではなく「そのコールバックがすでに実行された」に依存するかです。2 つの境界があります。テストのタイムアウト予算は失敗を待つものであり、タイミング依存の正しさではありません。ユーザー可視の処理のペースを整えるだけの意図的な遅延はこのルールの対象外です。開発者側の例外はありません。タイミング技法が真に必要な場合はドメイン所有者に相談します。

```ts
// The order is written into the code: dispose the old store, then build its successor.
declare let store: { dispose(): void }
declare function createStore(scopeKey: string): { dispose(): void }
declare const scopeKey: string

store.dispose()
store = createStore(scopeKey)
```

```ts ignore-check
queueMicrotask(() => { if (!ready) refresh() })   // assumes the microtask runs before the next frame
setTimeout(() => setMounted(true), 0)             // waits for a render that was never sequenced
```

誤った形式は断続的に失敗します。落とされたリフレッシュや決して同期しない状態が、負荷の下でだけ再現します。

## オプション入力は両方の振る舞いを明示する

オプションのパラメーターは、それが存在するときと存在しないときに呼び出し先が何をするかを明示するため、呼び出し元は防御的演算子から一方を推測するのではなく、契約から両方の振る舞いを読みます。`?.` が積み重なることがシグナルです。それぞれが、そのフィールドがそもそも存在すべきかについての決定の代わりになっています。

判定条件は、`?.` が既知の値における本当にオプションのホップを処理するのではなく、フィールドが存在するという確定の代わりになっているかです。要件はインターフェース契約を拘束します。両方の分岐が戻り値と副作用を明示し、呼び出し元はその契約からそれらを読みます。例外のチャネルはありません。両方の振る舞いを明示できない呼び出しサイトは、スタイルの好みではなくレビューのブロッカーです。

```ts
// Both behaviors are in the contract: no options object means "unknown", not "skip".
interface ResolveInput { options?: readonly string[] }
type Resolution = { kind: 'unknown' } | { kind: 'value'; value: string }

function resolve(input: ResolveInput): Resolution {
  if (input.options === undefined) return { kind: 'unknown' }
  return { kind: 'value', value: input.options.join(',') }
}
```

```ts ignore-check
const value = a?.b?.c?.d ?? fallback   // four layers of fallback hide whether d should exist
```

誤った形式は契約を曖昧なままにし、エラーを飲み込むため、次の呼び出し元がその上にさらにフォールバックを追加します。

## 1 つの値は 1 つの事実を表す

未知の値、空のコレクションや文字列、そして本当に存在しないフィールドは 3 つの異なる事実であり、3 つの表現を取ります。1 つの `undefined` や `null` がそれらの複数を表すことはなく、それぞれに対して異なる動作をしなければならないコンシューマーは、境界でそれらを潰すのではなく明示的に判別します。

判定条件は、単一の「値なし」が複数のセマンティクス（まだ問い合わせていない、問い合わせたが空、判定できない）を運ぶかです。判別可能なユニオンや明示的な enum 値が通常の実装であり、[core.md](../subsystems/core.md#repo-wide-type-patterns) で型が定義されている場所に文書化されています。

```ts
type ValueState<T> = { kind: 'unknown' } | { kind: 'empty' } | { kind: 'value'; value: T }
```

```ts ignore-check
return null   // means "not queried", "queried, nothing there", and "cannot tell", all at once
```

誤った形式は呼び出し元にケースを区別できなくさせるため、UI の状態、キャッシュ・ポリシー、リトライの決定がすべて誤ります。

## 新しい active/current シングルトンは拒否される

新しいグローバルな `active` または `current` シングルトンは導入せず、既存のものは移行して除去します。グローバルに一意な選択は、このコードベースが行わない設計上の表明です。1 つのデータの複数のビューが、それぞれ自身の参照または key を保持します。

判定条件は、変更がグローバルに一意な選択状態フィールドを導入するかです。置き換えの形は、1 つのデータとその上の複数のビューであり、各ビューが自身の参照を持ちます。既定の例外はありません。概念がサービスごとに本当に 1 回だけ存在する場合、`inject` を通じて表現されるサービスまたはインスタンスのフィールドがそれを運び、モジュール・レベルのフィールドが運ぶことは決してありません。

```ts ignore-check
// Each view holds the key it renders; nothing global decides what is "current".
function ConversationView({ sessionKey }: { sessionKey: SessionKey }) { /* … */ }
```

```ts ignore-check
let activeTabId: TabId | undefined   // module-level; every component reads the same one
```

誤った形式は独立したインスタンス同士を干渉させ、どのビューがフォーカスを所有するかをレビュアーが監査できなくなります。

## 共有状態には 1 つの所有者がいる

共有状態はサービスまたはインスタンスにぶら下がるため、その所有者はライフサイクルを持つ名前付きオブジェクトです。モジュール・レベルのキャッシュ、シングルトン、グローバルな `WeakMap`、または `Symbol.for` / `globalThis` / `window.__X__` の迂回は、その所有者を隠し、すべての teardown を超えて生き残ります。

判定条件: 値を所有するサービスまたはインスタンスを名指しできるか。所有者を名指しできない場合、実装の前に所有者を決めます。境界は、サービスまたはインスタンスのフィールドを任意の数の注入されたコンシューマーが読んでよく、その値をドメイン間で渡す場合はグローバル名前空間ではなく既存のインターフェースを通るということです。既定の例外はありません。グローバルなルックアップ・キーが真に必要な場合は、その回収ポリシーとともにドメイン所有者へ相談します。

```ts
// The shared value hangs on the session service instance; consumers inject it.
interface Sessions { readonly current: string }
declare const ctx: { get(name: 'sessions'): Sessions }

const sessions = ctx.get('sessions')   // one owner, lifetime tied to the service
```

```ts ignore-check
globalThis.__dshCache = new Map()   // shared across instances, reclaimed never
```

誤った形式はインスタンス同士を上書きさせ、回収のタイミングを未知にし、テスト間で状態を漏らします。

## ビジネス・ケイパビリティは自身を所有する

ビジネス・ケイパビリティは自身のパッケージに存在し、登録サーフェス（レジストリ、`slots.inject`、ウォーターフォール、イベント購読）を通じて接続します。runtime、controller、manager、gateway は汎用的なオーケストレーションを運びます。レジストリ構造と破棄のライフサイクルを保持してよく、エントリーはそれらを所有するパッケージが書き、ビジネス項目の中央テーブルをハードコードすることは決してありません。

新しい区別、フィールド、アダプターが必要かどうかを所有者を選ぶ前に確定してください。不要な概念を移しても、その保守コストは消えません。必要な振る舞いについては、ローカル実装が複雑さをほとんど加えない場合、そのプロデューサーやコンシューマーの近くにある既存の拡張ポイントを優先します。プロデューサーは、共有型を宣言するパッケージを編集せずに、自身のパッケージ内で[宣言マージ](../subsystems/core.md#the-map--derived-union-pattern)を通じて機能バリアントを宣言できます。すべてのコンシューマーに共通の義務は共有の所有者に属します。

判定条件は、ケイパビリティが自身を宣言して登録するか、中心レイヤーがビジネスの決定を行うのではなく転送と寿命の管理だけを行うかです。境界は、中心レイヤーがレジストリ構造と破棄のライフサイクルを保持し、エントリーは所有するパッケージから来るということです。1 つの例外があります。純粋なインフラストラクチャ（ロギング、トランスポート、ストレージ）はビジネス登録サーフェスを通じて接続しません。中心レイヤーの新しい分岐は、なぜ所有パッケージに置けないかを説明しなければなりません。

```ts
// The business package registers its own slot and handlers in its apply.
interface Slots {
  inject(name: string, register: () => void): void
  register(name: string): void
}
interface Context { readonly slots: Slots }

export function apply(ctx: Context): void {
  ctx.slots.inject('conversation.footer', () => ctx.slots.register('conversation.footer'))
}
```

```ts ignore-check
if (pluginName === 'ui-attachment') { /* business branch inside the manager */ }
```

誤った形式は、新しいケイパビリティのたびに中心レイヤーを編集させるため、そこが変更のボトルネックと競合のホットスポットになります。

## 変更はパフォーマンスのエビデンスを運ぶ

ホット・パスのコストを動かし得る変更は、触れたコードを前後で測定し、その数値を条件とともに報告します。3 つの指標は runtime、ピーク・メモリ、定常状態メモリで、1 つの環境、1 つの入力で測定します。粒度は変更されたパスであり、マシン全体のスコアではありません。「全体として遅くなっていない」という報告はエビデンスではなく、別の場所で改善したパスがここのリグレッションを免除することもありません。最適化された first paint でも `append` に対して説明する必要があります。

判定条件: 1 つの環境と 1 つの入力の下で、3 つの指標のいずれかが変更されたパスで有意に悪化していること。

ベースラインは、測定対象の変更を除いた同じパスです。通常は測定フックを持たないネイティブ実装であり、以前の実装ではありません。両側に同じフックを付けて古い実装と新しい実装を比較すると、差が薄まって消えます。両側が同じオーバーヘッドを払うため、両方が遅いのに測定は「変化なし」と報告します。

測定は [benchmarks/](https://github.com/deepseek-ai/deepseek-harness/blob/master/benchmarks/AGENTS.md) が所有し、測定対象のユーザー・パスごとに 1 つのディレクトリを持ち、そのシナリオがエントリー・ポイントです:

```sh
# Run the owning scenario before and after the change; the numbers go into the PR.
pnpm run test:bench
```

パッケージ・ローカルの診断は `.perf.ts` 接尾辞を付けて所有者のそばに留まり、`test:bench` には参加しません。CI ゲートは `pnpm run check:ci:bench`（ジョブ `node 24 / benchmarks`）で、スケールされたレポートを発行します。そのゲートはアラートであり、合格ではありません。そこでのリグレッションは、所有するシナリオを再実行し、セグメントをローカルでプロファイルして、どの部分が遅くなり、どの変更がそれを導入したかを見つけて特定します。保持メモリの主張には `pnpm run test:bench:memory 4g|8g` を使い、これはリストされたファイルを、swapless cgroup 内の変更されていない予算の下で再実行します。

境界: runtime パスに触れない編集（コメント、ドキュメント、テスト専用コード）は測定を必要とせず、プロダクションのパフォーマンス・コードは、都合よく削除すべきベンチマーク計装ではありません（`performance.now()` と `scheduler.yield()` はプロダクションのデコードとエンコードのチャンク・スケジューリングの一部です）。性能を真に犠牲にするアーキテクチャ変更は、測定されたコストと理由を明示し、ドメイン所有者が承認します。「レイヤリングがそれを必要とした」は自己都合の理由ではありません。コマンド、数値、環境、入力サイズを記録し、ベースラインを再構築できるようにします。

```text
CI benchmark reported green, so the change is accepted as performance-neutral.
```

その形式は、ベンチマークがモデル化していないパスに実際のリグレッションを出荷します。

## 移行は振る舞いを保つ

移行やパッケージ分割は、外部から観察可能な振る舞いを等しく保ちます。エラー・コード、診断、データフローの順序、UI のタイミングを項目ごとに比較します。コードの移動は削除と書き換えより優先され、書き換えはセマンティクスがどのみち変わるコードに適し、その場合は変更が差分を列挙します。パッケージやエクスポート・パスの変更は振る舞いの変更ではなくキャリアの変更であり、振る舞いの変更は独自の変更です。

判定条件は、前後での観察可能な振る舞いの項目ごとの比較です。完全な等価が不可能な場合、変更は異なる項目とそれぞれの検証方法を明示し、ドメイン所有者が承認します。移行中に見つかった欠陥は、移動に折り込むのではなく独自の変更として修正します。

```sh
# The move keeps paths and error codes; the reference updates are the whole diff.
git mv packages/old/src/endpoint.ts packages/new/src/endpoint.ts
```

```ts ignore-check
// The migration silently upgrades the error handling it was only supposed to move.
throw new EndpointError('not found')   // was: return { ok: false, code: 'not_found' }
```

誤った形式は振る舞いの変更を移動の中に埋めるため、レビューが移転とセマンティクスを分離できず、リグレッションを特定できなくなります。

## Dispose は要求するだけでなく、静止状態に到達しなければならない

kill または abort した後、子の終了を await します。オプションの通知は早く停止してかまいませんが、完了や最終永続化に必要なリソースとリスナーは、コンシューマーが落ち着くまで生き残らなければなりません。[Cordis の所有チェック](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/cordis-styling-nitpicking/SKILL.md#p0-cleanup-dependency-released-before-consumer-completion) を参照してください。

判定条件は、teardown が実際の静止を await するか、それを要求するだけかです。境界は、await すべき所有者のない処理（切り離された未観測のタスク）であり、teardown はそれを追跡する義務を負いません。コンポーネントが開始し、まだ観測できるものは、完了を報告する前にすべて await しなければなりません。

## ディスパッチャーでコールバック例外を封じ込める

各リスナーの例外を捕捉して報告し、ディスパッチの promise を reject したり、後続のリスナーが実行できなくなったりしないでください。リスナーの所有者は、[エラー処理のルール](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/dsh-error-handling/SKILL.md#catch-recover-or-stop) の下で自身が変更する状態を処理しなければなりません。捕捉だけでは状態を復元しません。

判定条件は、throw するリスナーがディスパッチの promise を reject したり、後続のリスナーの実行を妨げたりできるかです。境界は呼び出し元自身の await であり、ディスパッチャーがそれを要求した操作へ意図的に伝播するエラーは封じ込めの失敗ではありません。

```ts
async function dispatch(listeners: ((event: string) => Promise<void>)[], event: string): Promise<void> {
  for (const listener of listeners) {
    try { await listener(event) } catch (error) { console.warn('listener failed', error) }
  }
}
```

## 信頼できない出力元にアンビエント環境や予測可能なパスを渡さない

spawn されたコマンドは洗浄済みの env を受け取ります（`*KEY*`/`*SECRET*`/`*TOKEN*`/`*PASSWORD*` を落とす）。harness の資格情報が出力、`env`、spill ファイルへ漏れないようにするためです。一時ファイルや spill ファイルは、プライベート（0700）ディレクトリ、ランダムな名前、所有者専用の排他オープン（`'wx'`、`0o600`）を使います。予測可能で誰でも読めるパスは、シンボリックリンク競合と情報開示を招きます。

判定条件は、spawn されたプロセスがアンビエント環境を継承するか、別のプロセスが予測して置き換えられる場所へ書き込むかです。境界は、外部入力を持たない信頼されたインプロセス・コードであり、このルールは統治しません。

## リンク形状のパスは unlink する

シンボリックリンクや Windows junction の可能性があるパスは、`lstatSync().isSymbolicLink()` の後に `unlinkSync` で削除します。unlink はリンクだけを削除し、実ディレクトリを拒否するため、リンクをたどってターゲットに入ることは決してありません。Windows の `rmSync(link)` は junction で `ERR_FS_EISDIR` を throw し、再帰削除は junction を通ってターゲットへ降りることがあります。再帰的な `rmSync` は既知の実ディレクトリに取っておきます。

判定条件は、削除プリミティブがリンクをたどってターゲットに入り得るかです。境界は、コード自身が所有するディレクトリ内で作成したパスであり、単純な再帰削除がすでに安全と分かっている場合です。
