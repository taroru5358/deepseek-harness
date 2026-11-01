# 公式 DeepSeek LLM API のワイヤー拡張

日本語 | [English](../../docs/deepseek-llm-api-wire-extensions.md) | [中文](../../docs/deepseek-llm-api-wire-extensions.zh.md)

> これは非公式の日本語訳です。英語版（[docs/deepseek-llm-api-wire-extensions.md](../../docs/deepseek-llm-api-wire-extensions.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

このリファレンスは、`deepseek-official` の Messages リクエストで [`@deepseek-ai/dsh-llm-deepseek`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm-deepseek/README.md) が送る、DeepSeek Harness 固有の HTTP ヘッダーと追加の JSON フィールドをすべて定義します。上流の DeepSeek API が所有するフィールドを再定義するものではありません。プロバイダー中立の LLM インターフェースと `llm-pi-ai` は、これらの追加を実装しません。

アダプターは、設定されたゲートウェイも含め、解決済みの `baseURL` へ追加分を送ります。それらは `messages`、システム・プロンプト、ツール・スキーマの外に留まるため、モデル入力のトークンを増やさず、モデルから見えるプレフィックスも変えません。

## ワイヤーの名前空間とバージョニング

| 場所 | 命名 | 例 |
|---|---|---|
| HTTP フィールド名 | 小文字のケバブケース。HTTP のマッチングは引き続き大文字小文字を区別しない | `user-agent`、`x-deepseek-harness-session-id` |
| DeepSeek リクエスト・ボディの拡張フィールド | 予約された `dsh_` プレフィックスを伴うスネークケース | `dsh_plugin_packages`、`dsh_session_log` |
| DSH 所有のネストした JSON メンバー | キャメルケース | `afterSeq`、`throughSeq`、`sessionId` |
| タグ付きの値 | ケバブケースの文字列。永続イベントは `domain/action` を使う | `session-log-deepseek/delivery-accepted` |

各ボディ拡張は、自身の `version` を独立に所有します。バージョンはそれを含むオブジェクトにのみ適用され、異なるフィールドのバージョン間に互換性や順序の関係はありません。JSON メンバーの順序はプロトコルの一部ではありません。

[`DeepSeekLlmApiExtensionRegistry`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/deepseek-llm-api-extensions/README.md) は、トップレベルの拡張名ごとに 1 つのプロバイダーを予約します。空の名前や空白で埋められた名前、重複した登録、ベースの DeepSeek リクエストとの衝突は、HTTP ディスパッチの前に失敗します。

## リクエスト・ヘッダー

| ヘッダー | 存在 | 値 |
|---|---|---|
| `user-agent` | Files API の操作を含む、すべてのプロバイダー HTTP リクエスト | `product/version (+url)` 形式のアプリケーション識別子。既定の product は `deepseek-harness` |
| `x-deepseek-harness-user-id` | 認可されたすべてのモデル・リクエスト | 解決済みの Harness home に対する、安定した匿名 UUID |
| `x-deepseek-harness-session-id` | Session id を伴うモデル・リクエスト | リクエストの `sessionId` 文字列そのもの |
| `x-deepseek-harness-compact` | 目的が `compaction` であるモデル・リクエスト | リテラル文字列 `1` |

認証情報の失敗は匿名ユーザー id の解決より前に起きるため、認可されていないリクエストはこれらのヘッダーを送らず、識別子のファイルも作成しません。Session を持たない直接のリクエストは `x-deepseek-harness-session-id` を省略します。Session タイトルと Auto review のリクエストには追加の目的ヘッダーがなく、`sessionId` を運ぶ場合には通常の Session id のルールが引き続き適用されます。

## ボディ拡張のトランザクション

アダプターは、登録済みプロバイダーにフィールドの準備を求める前に、完全なベース・ボディを、正確な `messages` を含めてシリアライズします。プロバイダーは、その不変のボディ、リクエストのキャンセル・シグナル、省略可能な `sessionId` と補助呼び出しの `purpose` を受け取ります。`undefined` を返すと、そのプロバイダーのフィールドはそのリクエストから省略されます。

準備された JSON 値はプロバイダー所有の状態から切り離され、ベース・フィールドのトップレベルの兄弟としてマージされ、同じ HTTP ボディでシリアライズされます。準備に失敗したコントリビューターはそのリクエストから省略され、最初の失敗だけがログに記録されますが、他のフィールドは引き続き送信されます。ベース・フィールドと衝突するフィールドはリクエストを妨げます。マージ後のボディのシリアライズに失敗した場合、アダプターは拡張フィールドを一切含まずにベース・ボディを送り、受け入れトランザクションをスキップして、コントリビューターが後のリクエストで状態を再送できるようにし、省略したフィールド名をログに記録します。レジストリを持たないコンポジションは、拡張されていないベース・ボディを送ります。

設定されたエンドポイントが HTTP 2xx を返した後、アダプターは SSE レスポンス・ボディを読む前に、準備済みの `accept()` トランザクションを実行します。トランスポートの失敗と非 2xx のレスポンスは、どのコントリビューションも受け入れません。受け入れの失敗はログに記録され、モデル・リクエストを失敗させません。コントリビューターは受け入れられなかった状態を後のリクエストで再送します。受け入れはエンドポイント・レベルの HTTP 成功を記録するものであり、SSE ストリームが完了したことや、エンドポイントが拡張を永続化したことを主張するものではありません。

## `dsh_plugin_packages`

[`@deepseek-ai/dsh-plugin-package-inventory-deepseek`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/plugin-package-inventory-deepseek/README.md) が、Loader に裏付けられたアクティブなプラグイン・パッケージの完全な目録を提供します。このフィールドは既定で有効です。

```json
{
  "dsh_plugin_packages": {
    "version": 1,
    "packages": [
      {
        "name": "@deepseek-ai/dsh-example",
        "version": "0.1.1-rc.2"
      }
    ]
  }
}
```

| メンバー | 型 | 意味 |
|---|---|---|
| `version` | `1` | `dsh_plugin_packages` のスキーマ・バージョン |
| `packages` | 配列 | このリクエストに対するアクティブな集合の全体 |
| `packages[].name` | string | 所有する manifest からの、空でない正確な npm パッケージ名 |
| `packages[].version` | 省略可能な string | 同じ manifest からの、空白でない正確なパッケージ・バージョン。利用できない場合は省略 |

すべてのリクエストは、ホスト・ツリーからアクティブな非グループの Loader エントリーを読み直し、リクエストの Session に対して利用可能な場合は、その常設の agent-preset ツリーも読み直します。相対および絶対のモジュールは最も近い所有 manifest を使い、ベアのパッケージ・エントリーは、それをアクティブ化した Loader の解決ベースに従います。空白でない文字列のバージョンを持たない名前付き manifest は、`private` にかかわらず `{ name }` を提供します。名前の欠落と読めない manifest は、影響を受けるエントリーだけを省略します。パッケージのメタデータがリクエストを妨げることはありません。

送信側は正確な `(name, optional version)` の組を重複排除し、まず `name`、次に `version` でソートします。比較はロケール非依存のテキスト比較で、バージョンのないものを先にします。同じパッケージの同時にアクティブなバージョンは、名前だけのエントリーも含めて別々のエントリーのままです。レシーバーはバージョンの欠落を受け入れなければならず、配列をパッケージ名でまとめてはならず、配列の順序からパッケージのアクティブ化を推測してはなりません。

無効、保留中、失敗、アンロード中、dispose 済み、構造上の Loader エントリーは存在しません。通常の依存関係、名前付きの所有パッケージを持たない緩いモジュール、プログラムからマウントされた子 fiber、インメモリの動的プラグインも、Loader に裏付けられた権威あるパッケージ識別子を持たないため存在しません。

該当するエントリーのない有効な目録は `packages: []` を送り、コントリビューターを無効にすると `dsh_plugin_packages` フィールド全体が省略されます。パッケージの識別子はプロバイダーのメタデータであり、モデル入力に入ることはありません。

## `dsh_session_log`

[`@deepseek-ai/dsh-session-log-deepseek`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-log-deepseek/README.md) が、正準 Session ログの 1 つの連続したサフィックスを提供します。このフィールドは既定で有効です。ライブの Session と少なくとも 1 つのイベントを持つリクエストに適用され、直接のリクエスト、古い Session id、空のログ、登録済みのアップロード除外に一致するリクエストはフィールドを省略します。最初の保留イベントだけで `maxBytes` を超える場合や、シリアライズできない場合も同様です。コンポジションは `enabled: false` で無効にします。以下の例は、ワイヤー・フィールドを説明するためだけに論理 Session 形式 2 を使うものであり、[現在のライター形式](session-format-status.md)を示すものではありません。

```json
{
  "dsh_session_log": {
    "version": 1,
    "sessionFormatVersion": 2,
    "session": {
      "version": 2,
      "id": "session-id",
      "createdAt": 1780000000000
    },
    "afterSeq": -1,
    "throughSeq": 0,
    "events": [
      {
        "type": "turn/start",
        "seq": 0,
        "time": 1780000000001,
        "data": {
          "turn": 1
        }
      }
    ]
  }
}
```

| メンバー | 型 | 意味 |
|---|---|---|
| `version` | `1` | `dsh_session_log` のスキーマ・バージョン |
| `sessionFormatVersion` | 非負の整数 | このサフィックスが表す Session 形式の世代 |
| `session` | オブジェクト | 現在の Session ヘッダーの不変のワイヤー・プロジェクション |
| `afterSeq` | 整数 | このリクエストの前に受け入れ済みとして記録された最大のシーケンス。なければ `-1` |
| `throughSeq` | 非負の整数 | このリクエストが表す最大のシーケンス |
| `events` | 配列 | `afterSeq + 1` から `throughSeq` までの連続したイベント |

最初のアップロードは `afterSeq: -1` を使い、以降の各アップロードは同じ Session id の受け入れ済みの最大ウォーターマークの後から始まります。各アップロードは、その地点からの最長の連続した範囲を運び、そのシリアライズされたフィールドが `maxBytes`（既定で 8 MiB、ヘッダーと数値フィールドを含む UTF-8 バイト数）に収まります。そのため、上限を超えるバックログは複数の受け入れ済みリクエストにまたがります。送信側はイベント配列をリクエストごとに 1 回スナップショットし、その後の追加は後のリクエストに属します。

### ワイヤーの Session ヘッダー

`session` メンバーは、論理 Session のメタデータを生の JSON プリミティブに投影します。seeded な Session は、正確な `Session.inheritedEventCount` を `seedLength` として送り、seeded でない Session はそのフィールドを省略します。論理の `isSeeded` フラグはこのワイヤーに現れません。外側の `dsh_session_log.version` がこの拡張スキーマを選び、`session.version` が論理 Session 形式を選びます。Session ヘッダーのプロジェクションを変えるには、埋め込まれた論理形式も同時に変わる場合でも、拡張スキーマのバージョンを上げる必要があります。

| メンバー | 存在 | 意味 |
|---|---|---|
| `version` | 必須 | `Session.header` からの論理 Session 形式のバージョン。[形式のステータス](session-format-status.md)を参照 |
| `id` | 必須 | 正確な Session id |
| `createdAt` | 必須 | 非負の安全な整数による Unix エポック・ミリ秒 |
| `cwd` | 省略可能 | Session の作成時に記録された絶対の作業ディレクトリ |
| `parentSession` | 省略可能 | fork の親 Session id |
| `seedLength` | seeded な Session のみ | 正確な継承イベント数。空の継承プレフィックスでは 0 を含む |
| `origin` | 省略可能 | サブ agent の子に対するリテラル `subagent` |
| `delegationDepth` | 省略可能 | 非負の、永続化されたサブ agent の委任の深さ |
| `agentPreset` | 省略可能 | この Session の構成に使われた agent プリセットの id |

### 正準イベントのエンベロープ

各 `events` 項目は、他のすべてのリクエスト・フィールドとは独立に、正準イベントを生の JSON プリミティブとして運びます。すべてのイベントは `type`、数値の `seq`、`time`、`data` を含み、存在する場合は `ignorable: true` を保ちます。サーフェス・イベントは `surfaceOp` を必要とし、置換の範囲は数値の `startSeq` と `endSeq` を使います。システム、ユーザー、ツールのイベントは、数値の `sourceEventSeqs` も運ぶことがあります。Assistant のプロバイダー・メタデータは、埋め込まれたストリーム内に残ります。既知のログ専用イベントはサーフェスのメタデータを省略し、復元された未知の ignorable な記録は、サーフェス操作として扱わずに不透明なメタデータを保ちます。

### 受け入れウォーターマークと at-least-once 配信

エンドポイントが HTTP 2xx を返した後、コントリビューションは次の正準イベントを同じ Session に追加します:

```json
{
  "type": "session-log-deepseek/delivery-accepted",
  "seq": 8,
  "time": 1780000000002,
  "data": {
    "sessionId": "session-id",
    "sessionFormatVersion": 2,
    "throughSeq": 7
  }
}
```

`delivery-accepted` は、それを含む LLM リクエストに対して設定されたエンドポイントが HTTP 2xx を返したことを意味します。SSE の完了やリモートの永続化を主張するものではありません。イベントの `throughSeq` は先行するイベントを識別しなければならず、その `sessionId` はサフィックスが送られた Session を識別し、`sessionFormatVersion` はウォーターマークをその正確な論理世代に結び付けます。存在しない場合は履歴形式 v0 を意味します。

送信側は、現在の Session id と形式世代について一致する最大の `throughSeq` を畳み込みます。そのため、同時に受け入れられたリクエストがカーソルを後退させることはなく、別の世代のウォーターマークが現在のサフィックスを認可することもありません。再開されたプロセスは、永続ログからカーソルを再構築します。fork は、別の Session を名指す継承ウォーターマークを無視するため、子の id で進む前に、`maxBytes` で区切られたリクエストで自身の継承プレフィックスをシーケンス 0 からアップロードします。ウォーターマーク・イベント自体は、次の未送信のサフィックスに属します。

トランスポートの失敗、非 2xx のレスポンス、シリアライズ失敗後に拡張フィールドなしで送られたリクエストは、ウォーターマークを追加しません。エンドポイントの受け入れ後、ローカルへの永続化前にクラッシュすると、すでに受け入れられた範囲を再送することがあります。不確実性は重複を生みますが、シーケンスの欠落は生みません。独立したアップロード・ストアや切り詰めの経路はなく、`maxBytes` が各リクエストのフィールドを制限します。

## 露出とレシーバーの要件

リクエスト・ヘッダーは、Harness アプリケーションのバージョン、1 つの匿名の Harness home 識別子、省略可能な Session 識別子を露出します。`dsh_plugin_packages` は、アクティブな npm パッケージの名前とバージョンを露出します。コンポジションが無効にしない限り、`dsh_session_log` は Session の作業ディレクトリ、システム・プロンプトのスナップショット、ユーザーと Assistant の内容、埋め込まれた Assistant のストリーム、失敗した試行の出力、ツールの引数と結果、compaction の要約、フィードバック、プラグイン所有のイベントを露出することがあります。アダプターの API キーは Session イベントではないため、このフィールドに入りません。`baseURL` で選択されたゲートウェイは、公式エンドポイントと同じ値を受け取ります。

レシーバーは拡張フィールドを名前で扱い、各フィールドを自身の `version` でディスパッチし、異なるパッケージ・バージョンを区別し、JSON メンバーの順序を無視します。Session ログのレシーバーは、イベント型を解釈する前に連続したシーケンス範囲を検証します。`ignorable: true` を持たない未知の正準イベントは、ロスレスな再構築を妨げます。ベース・リクエストは、レジストリや特定のコントリビューションがなくても引き続き使用でき、フィールドの欠落は、そのコントリビューションがそのリクエストに適用されなかったことを意味します。Session ログのバックログが排出されている間、`throughSeq` は Session の最新イベントより遅れるため、2xx はレシーバーが現在のログを保持していることを示しません。
