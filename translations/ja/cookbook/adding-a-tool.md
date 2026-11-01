# ツール作成リファレンス

日本語 | [English](../../../docs/cookbook/adding-a-tool.md) | [中文](../../../docs/cookbook/adding-a-tool.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cookbook/adding-a-tool.md](../../../docs/cookbook/adding-a-tool.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

モデル向けツールが満たすべきコントラクトのリファレンスです。順を追って最初のツールを作る場合は [ツールを構築する](../user/develop/basic/tool.md) に従ってください。`packages/shell/tool-bash` が本番品質の3パッケージ構成の例です。

## 最小の形

```ts
import { readFile } from 'node:fs/promises'
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'my-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'read_file',
    description: 'Read a file from disk.',          // what the model sees
    parameters: {
      path: { type: 'string', required: true, description: 'Absolute path' },
      limit: { type: 'number' },                     // optional by default
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args, exec) {
      // args is TYPED from the schema: { path: string; limit?: number }
      // exec carries immutable identity + token; signal is the operational field
      return readFile(args.path, { encoding: 'utf8', signal: exec.signal })
    },
  }))
}
```

登録は効果ベースです。プラグインの fiber を破棄するとツールが登録解除されます。スキーマは自動的にシステム・プロンプト・アセンブリへ流れ込みます。

## execute() コントラクトのルール

- **引数は検証済みです。** `defineTool` は `execute` の実行前に、モデル生成の `arguments` を統合 `ParameterSchemaSpec` に対して検証します（型、必須キー、リテラル制約、exact-one union、ネスト値）。そのため `execute` 内の引数は `InferArgs` と一致します。明示的なオブジェクト・ノードは `additionalProperties: true | false` を宣言し、暗黙のパラメーター・ルートは開いたままです。DSL が表現しない制約（空でない文字列、正の数、フィールド間ルールなど）は引き続き手動でチェックします。直接登録された生の JSON-Schema ツールは、自身の入力検証を所有します。
- **登録は読み取り専用の定義を借用します。** 型付きの同一プロセス内コントリビューションはシリアライゼーション境界ではありません。登録後にスキーマを変更したりコールバックを置き換えたりしてはいけません。`schemas()` は明示的なモデル向け projection だけを実体化します。ツールをホットスワップするには、所有する効果を破棄して代替を登録します。コールバックのクロージャ内の可変状態は、通常のプラグイン状態のままです。
- **実行の同一性は保護されます。** レジストリは `arguments` を1回の再帰パスでデタッチされたロスレス JSON として実体化し、ポリシー開始前にその値を凍結し、不透明な `exec.token` を割り当てます。`callId`、`name`、`arguments`、`agent`、`token`、呼び出し元が所有する必須の `signal`、任意の外側トランスポートの `parent` トークンは、ディスパッチを通じて不変のままです。`parent` は同一性専用で、ライブな外側実行を公開しません。`args` は読み取り専用の入力として扱います。可変ビューを受け取るのはディスパッチを包むラッパーだけであり、期限を課すために必須の `exec.signal` を置き換えて復元できますが、削除はできません。
- **1つの正規 JSON 値を宣言して返します。** `output.schema` は `ValueSchemaSpec` を使い、object、array、scalar、null のルートを持てます。`execute` は推論された値だけを返します。レジストリはそれをロスレス JSON としてスナップショットし、検証し、凍結し、`output.render(args, value)` に渡します。本体からコンテンツ・ブロックを返したり、呼び出し元に id やフィールドを散文から解析させたりしてはいけません。
- **throw または無効な値の返却は `isError` を意味します。** レジストリは throw を捕捉し、オブザーバーが実行される前にスキーマ、レンダラー、メタデータ・プロジェクター、ロスレス JSON の失敗を封じ込めます。インフラストラクチャの失敗には throw してください。理想ではない状態（ゼロ以外のプロセス終了など）を Native レンダラーが説明する場合でも、成功したドメイン結果は正規値で表現します。
- **`exec.signal` を尊重します。** 発火したら進行中の作業をキャンセルします。
- **永続カード・データを `presentationMeta` で投影します（任意）。** `output.presentationMeta(args, value)` は同じ正規値からリプレイ可能な JSON を導出します。コアはそれを `tool/result` に永続化し、`presentResult` に渡します。そのため、結果時の事実（`write`/`edit` で適用されたハンクなど）を必要とするカードは、正規値を永続化せずにリプレイを生き延びます。ネストされた Code ディスパッチにはカードがないため、プロジェクターはスキップされます。
- **非同期通知には `exec.agent` を使います。** `agent.inject({ content, source: { kind: 'my-plugin' } })` は、次のモデル・リクエストが見る永続コンテキストを追加します。ウェイクアップではありません（idle のエージェントは idle のままです）。プラグイン内で [`MessageSourceMap`](../subsystems/llm-streaming.md#content-blocks-and-messages) を通じてその `kind` を宣言してください。Session 形式 V4 は、引退した `{ kind: 'plugin', plugin: '<name>' }` ラッパーをメッセージ受け入れ時に拒否します。破棄されたエージェントに対してガードしてください（try/catch）。

## 長時間実行される作業

`run_in_background` をプロデューサー設定でゲートし、`ctx.jobs.start({ kind, label, owner: exec.agent, run })` を通じて登録します。レジストリはプロデューサー本体の前に事前中断された呼び出しを拒否します。ランタイムは `run()` が作業を開始する前に所有権とタスク・コントローラーの可用性を検証し、その後 id、セッション・フェンス、汎用制御ツール、通知、所有者クリーンアップを提供します。成功したバックグラウンド分岐は `{ kind: 'background', jobId }` のような型付き正規ハンドルを返します。その Native レンダラーは `started background job bash-1` のような人間向け散文を保ってもかまいませんが、PTC モードはその散文を解析して id を復元してはいけません。

仕様は、同期の `cancel`、リソース・クリーンアップ後に決着する拒否しない `done`、およびレジストリがジョブの出力リングへ送り込む pull 型 `output` ソースか、スターターが受け取る `JobHandle` を通じた push のどちらかを提供します。`dsh-tool-jobs` はそのリングからモデルの消費読み取りをレンダリングします。事前中断された呼び出しは失敗です。成功出力スキーマを満たす id を持つタスクが存在しないためです。`ctx.jobs.start()` が id を公開した後は、`exec.signal` ではなくタスク所有のキャンセル信号を使います。後からの外側呼び出しのキャンセルは呼び出しの待機を止めますが、公開済みの作業を終了させません。`job_kill`、所有者の破棄、サービスの teardown がその寿命を所有します。フォアグラウンド作業は `exec.signal` に結合されたままです。[バックグラウンド・ジョブ・ランタイム・リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/jobs/jobs/README.md)と、ストリーム・プロデューサーの `dsh-tool-bash` を参照してください。

## 実行ポリシーと観察

デプロイ・ポリシーをツールに組み込まないよう優先してください。拡張可能な allow/deny/ask ポリシーには `tools/pre-execute`（[パーミッション・ゲートの例](extension-cookbook.md#a-hook-plugin-permission-gate-example)）、後続リスナーが取り消せない最終的な単調拒否には `ctx.tools.guard()`、期限、リトライ、メトリクス収集でディスパッチを包むには `tools/execute`、プレゼンテーション・コンテンツや返却値の置換、結果のブロック、モデル向けコンテキストの付加には `tools/post-execute`、不変の正規化された結果の観察には `tools/result` を使います。コンテンツの置換は `value` へのプログラム的アクセスをそのまま残します。機密性ポリシーは値をブロックまたは置換します。sandbox 実装はツールの executor 実装内で実行することもできます。[`dsh-tools` README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tools/README.md#extension-points)が各拡張ポイントの入力、順序、戻り値、失敗時の振る舞いを定義します。

## PTC モードはツールにそのまま到達する

[PTC モード](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/tools/README.md)では、可視の登録済みツールがすべて `await tools.<name>(args)` として追加の統合なしで利用できます。生成された `ToolArgsMap` と `ToolOutputMap` は同じスキーマから正確な引数型と正規戻り値型を導出し、呼び出しは通常の実行パイプラインに再入します。成功した呼び出しは、レンダリングされた Native コンテンツではなく、ポリシー後の最終的な正規 JSON 値に解決します。失敗した呼び出しは実際の `ToolCallError` で拒否されます。プログラムが検査できるのはその `name`、`toolName`、人間可読な `message` だけで、内部エラー・コードや失敗 union は検査できません。

`output.schema` は有用なプログラム的 API として設計します。ハンドルとフィールドを直接返し、正直な値である場合は scalar/array/null ルートを許し、人間向けの説明は `output.render` に保ちます。中間値は実行ローカルであり、永続化もプロンプト切り詰めもされず、バイト上限もありません。そのためプロデューサーの正直な取得境界とプロセス・メモリは依然として重要です。設定可能な出力上限とモデル向けスピル・パイプラインを越えるのは、外側の `run_code` のログ／結果だけです。

## ツールが UI でレンダリングされる方法

ツールの `output.render` はモデル向けコンテンツを返します。その **UI カード**は別の関心事であり、純粋なプレゼンテーション projection と任意の `presentCall` / `presentResult` メソッドで宣言します。これらは正規値と並行して設計してください。UI プレゼンテーションを持たないツールは、汎用カード（タイトル = ツール名、入力 = 生の引数）にフォールバックします。

両方のメソッドは **`card` タグ付きの描画意図**を返します。ツールの動作に合うカード種別を選んでください:

- `presentCall(args)` → `ToolCallView`（PENDING カード）:
  - `{ card: 'generic', title, kind?, rawInput?, content?, locations? }` — 既定。アイコンには `kind` を設定し（`read`/`search`/…）、ツールが触れるファイルには `locations: [{ path, line? }]` を設定して、対応エディターが追従／ジャンプできるようにします。
  - `{ card: 'terminal', title, description?, cwd? }` — 呼び出しがシェル・コマンドそのものです。`title` がコマンドで、`description` はターミナル・カードの上にレンダリングされます。（tool-bash。）
  - `{ card: 'diff', title, diffs, locations? }` — 呼び出しがファイルを作成または変更します。`diffs: [{ path, oldText, newText }]`（新規ファイルなら `oldText: null`）がインライン diff カードとしてレンダリングされます。（tool-fs の `write`/`edit`。）
- `presentResult(args, { content, isError, meta? })` は完了済みカードを返します:
  - `generic` は任意のタイトルとコンテンツを提供します。
  - `terminal` は生の出力と任意の終了メタデータを提供し、各 UI が対応ビューまたはフォールバック・ビューをレンダリングします。
  - `diff` は適用済みハンクを提供し、多くの場合 `output.presentationMeta` で導出され、永続化された `result.meta` に運ばれてリプレイで再現されます。変更ツールは diff 結果を保ちます。完了ビューが保留カードを置き換えるためです。
  - `read` は永続化された `result.meta` から再構築された完了済みファイル・ウィンドウを提供します。ファイルの `path`、1始まりの `offset`、返された `lines`（それぞれファイル行番号を保つ）、`totalLines`、任意の `lang` ハイライト・ヒントです。`read` ケイパビリティを持たない UI は生の結果コンテンツにフォールバックします。`read` の呼び出しビューはありません。読み取り呼び出しの保留状態は汎用カードのままで、コンテンツは `execute` の後にのみ存在します。（tool-fs の `read`。）
  - `search` は永続化された `result.meta` から再構築された発見結果を提供します。ファイルごとにグループ化されたマッチ（`shape: 'matches'`、grep）またはフラットなパス・リスト（`shape: 'paths'`、glob）に加え、`truncated`/`total` を持ち、UI が上限付きの結果を完全なものとして提示しないようにします。ビューは結果テキストを運びません（検索カードを持たない UI は生の結果コンテンツにフォールバックします）。`search` の呼び出しビューもありません。発見呼び出しの保留状態は汎用カードのままで、マッチは `execute` の後にのみ存在します。（tool-fs-search の `grep`/`glob`。）
  - `web` は完了済みの web 取得を提供し、`kind: 'search' | 'fetch'`（構造化検索ソースまたはフェッチ・サマリー）で判別され、`result.meta` から導出されます。本文コピーは運ばないため、`web` ケイパビリティを持たない UI は生の結果コンテンツにフォールバックします。（tool-web の `web_search`/`web_fetch`。）

厳格なルール（破ると効いてきます）:

- **純粋性。** これらはライブ・ストリーミングとセッション・ログのリプレイの両方で実行されるため、`args`（+ 結果）の純粋関数でなければなりません。I/O なし、セッション状態の読み取りなし、クロック／乱数なしです。diff は引数から導出されます（呼び出し時のプレゼンターには以前のファイル内容がないため、`write` は `oldText: null` を使います）。セッション・コンテキストを供給するのはツールではなく UI アダプターです。`presentCall` 内でファイルの古い内容や作業ディレクトリを求めたくなったら、止めてください。それはプレゼンターではなく、永続的な結果メタデータまたはアダプターに属します。
- **UI 専用の整形はモデル結果に入れません。** フェンス付きの ` ```console ` ブロック、diff、相対化されたパス — これらは UI に提供するためだけに正規値や Native コンテンツへ入れるものではありません。`output.render` はモデル向け散文を所有し、`presentationMeta` とカード・プレゼンターがリプレイ可能な UI 状態を所有します。`terminal` の結果ビューは生の出力を運び、フォールバックの枠付けはアダプターが追加します。
- **`defineTool` は表示パスをソフト検証します。** 不正または古い記録済み引数では、ラッパーは throw せずに `undefined`（汎用フォールバック）を返します。表示がリプレイをクラッシュさせてはいけません。

中立的な語彙は `dsh-tools` にあります。ツールが UI やトランスポートの型を import することはありません。この API のコンシューマーは各 `card` を独自のビューにマップします。設計とその理由は[描画意図 union の Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-07-02-tool-render-intent-union.md)にあり、`dsh-tool-fs`（generic/diff）と `dsh-tool-bash`（terminal）が参照実装です。

## Web Client のプレゼンテーション

組み込み Web Client は `presentCall` や `presentResult` を消費しません。Session の `page` と `follow` は、永続化された `result.meta` を含む生の `tool/call` と `tool/result` イベントを転送します。Client プラグインはワイヤーのツール名を `tool.call.toolview` キー付きスロットに登録し、`ToolCallBlock` の引数、コンテンツ、エラー、メタデータ、既存の PTC ディスパッチ `parentCallId`、Session パスの事実からコンポーネント props を導出します。これらのワイヤー値をローカルで検証し、不正またはサポートされない入力には汎用行を返します。

既存の Web カードが、モデル向けコンテンツではロスレスに保持できない境界付きの構造化結果事実を必要とする場合は、`output.presentationMeta(args, value)` を使います。メタデータに React props や選択済みカードを保存したり、Host のツール実装をブラウザ・バンドルに import したり、別の Client プレゼンター・レジストリを作成したりしてはいけません。Host のプレゼンテーション・メソッドを定義するだけでは、専用の Web カードは追加されません。[Client 由来プレゼンテーションの Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-23-client-derived-tool-presentation.md)が所有権、フォールバック、等価性の要件を定義します。

## 検証

[リポジトリのテスト方針](../testing.md)と、所有パッケージのテスト・ドキュメントに従ってください。出荷されるモデルまたは UI に見える変更には、そこで指定された組み立て済みカバレッジが必要です。
