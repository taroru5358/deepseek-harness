# Web Client アーキテクチャ

日本語 | [English](../../../docs/subsystems/web-client.md) | [中文](../../../docs/subsystems/web-client.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/web-client.md](../../../docs/subsystems/web-client.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

Web Client は、独立してロードされるプラグインから組み立てられるブラウザー側の Cordis アプリケーションである。そのアーキテクチャには 4 つの再利用可能な基盤がある。[クライアント・モジュール](client-modules.md)がプラグイン・グラフをロードし、[API Gateway](../api-gateway.md)が型付きの Host 通信を提供し、[Slots](slots.md)が React UI を合成し、[Conversation](conversation.md)が Session 履歴のウィンドウをターゲット所有のビューに変換する。このページはこれらのシステムをつなぎ、Client モデルと機能パッケージがどこに属するかを定義する。

[キーボード・ショートカット](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/shortcuts/README.md)はウィンドウ・ローカルのコマンド登録と物理キーのディスパッチを所有し、[ショートカット・リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-shortcuts/README.md)は利用可能なコマンドとローカル入力アクションを提示する。コマンドの所有者はランタイム／プラットフォームごとのデフォルトを宣言し、既存のアクションをマウス制御と共有する。共有のモーダル・プリミティブが最上位レイヤーの Escape を調停し、フォーカスを復元する。

## レイヤーと所有権

| レイヤー | 主な所有者 | 責務 |
|---|---|---|
| Host アプリケーション | ビジネス・サービスと `packages/api/*-controller` の Host エントリー | 権威ある状態、永続化、変更の順序、アクセス・ポリシー、ストリーム生成を所有する。 |
| トランスポートと API のアセンブリ | `client/connection`、`api/gateway`、`api/remotes` | Client の世代を確立し、生成された `ctx.remote` メソッドとストリームを公開し、選択された Cordis イベントを転送し、キャンセルと結果を運ぶ。 |
| Client モデル | `api/session-controller/client`、`api/workspace-controller/client` | Host 状態の React 非依存のミラーを維持し、ストリーム／単項の競合を解決し、オブジェクト・アイデンティティとサブスクリプションを所有し、狭いコマンド・サービスを公開する。 |
| UI アダプター | `client/ui-session`、`client/ui-workspace` | モデルのオブザーバブルを、ルートまたは Provider に束縛された Session Slot のソースに変換し、ビュー・レベルのナビゲーションとステータス・ポリシーを所有する。 |
| Conversation データ | `client/ui-conversation`、`ui-chat` や `ui-trajectory` などのターゲット・パッケージ | 標準イベントと、圧縮された過去の Assistant 実行を、独立したターゲット・スナップショットに組み立て、共有の conversation シェルと入力フローを所有する。 |
| コンポジションとレンダリング | `client/ui-slots`、`client/ui-renderer`、`client/ui-layout`、機能 UI パッケージ | 拡張の場所を宣言し、コンポーネントの props を導出し、オブザーバブルを React フックに束縛し、最終的なツリーをマウントする。 |

依存の向きは Host 状態 → Remote トランスポート → Client モデル → UI アダプター → Conversation またはプレゼンテーション → Slots → React である。ユーザー・アクションは、注入された Client サービスまたは生成された Remote 名前空間をクローズオーバーするコールバックを通じて戻る。プレゼンテーション・コンポーネントは Cordis の `ctx`、トランスポート・オブジェクト、別の機能プラグインの実装を決して受け取らない。

## ブラウザー・ブート

Host は合成された `WebBootGraph` を `window.__DSH_BOOT__` に書き込み、パーサーがプリロードしたスクリプトが実行される前にブラウザー・モジュール・ローダーのファサードをインストールする。モジュール・システムは遅延 CommonJS テーブルである。バンドルをロードするとそのファクトリが登録され、エントリーを実体化すると、プラットフォーム・モジュールと宣言された動的依存関係に対する同期 `require` でファクトリが実行される。

Web ブート・カーネルはモジュール・システムを作成し、`immediately` のエントリーを先読みし、ベンダリングされた Cordis Loader をマウントし、すべてのグラフ・エントリーを作成する。Cordis のサービス注入がアクティブ化を決め、モジュール・グラフの順序は同期インポートを実体化できるかどうかだけを決める。完全な名簿が落ち着いた状態に達した後、`ui-renderer` はフレームワーク非依存のブート DOM をハイドレートし、唯一のコンテキスト・レベルの `renderSlot('root')` 操作を呼ぶ。[クライアント・モジュール](client-modules.md)はグラフ、バンドル・ルート、キャッシュ・リビジョン、ローダーの詳細を所有する。

## Remote 通信

Host のビジネス・サービスは、呼び出し可能なメソッドに Typert Remote デコレーターを注釈する。Host の生成は厳密な記述子、ランタイム・コーデック、宣言マージ、ソース・マップを出力する。Client 側の `api-remotes` アセンブリはそれらの生成された寄与を選択し、`ctx.remote.<namespace>` と Session スコープの `agentCtx.remote.<namespace>` の下に具体的なメソッドをマウントする。機能パッケージは生成されたサービス・フェイスに依存し、Gateway の実装や Host パッケージのランタイム・エントリーには依存しない。

Connection は要求 URL の解決、相関、`/api` キャリア、信頼チェック、正確な Fetch ルート、接続の世代を所有する。API Gateway は Remote のディスパッチ、キャンセル、論理ストリーム、選択された Host イベントの転送を所有する。コントローラー操作は生成された Remote メソッドまたは明示的な Remote ストリームに属し、機能所有のダウンロードは正確な Fetch ルートを登録する。[API Gateway リファレンス](../api-gateway.md)が生成と呼び出しを定義し、[Connection README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/connection/README.md)が物理キャリアと信頼ポリシーを定義する。

内部の `$events` 論理ストリームは Connection の世代ソースである。その開始 `ready` フレームはパス表示に使う Host ホームを運び、Host リスナーが接続された後に、どのコントローラーもベースライン読み取りを始める前に世代を確立する。`ctx.remote.$on()` は許可リスト済みの通常イベントをルート Client Context に、スコープ付きウォーターフォール・イベントを解決された Session Context に届ける。ウォーターフォール・リスナーは結果を返すか、`next()` を呼ぶか、拒否する。

## Client モデル

各 API コントローラー・パッケージは、対になる Host と Client のフェイスを所有する。Host 側は権威ある変更とストリーム生成を所有する。Client 側は同じ生成されたワイヤー型に基づく、アイデンティティが安定した React 非依存のモデルを所有し、オブザーバブルなスナップショットとコマンドを公開する。UI パッケージはこれらの Client サービスを消費し、コンポーネント・ストアにトランスポート状態を再現しない。

### Session

[`api/session-controller`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/session-controller/README.md)は、一覧、検索、作成、プロンプト、キュー、キャンセル、ページネーション、フォロー／制御ストリームのための Host コマンドを公開する。その Client 側は `ClientSessions → SessionManager → Session` として構成される。

- `ClientSessions` は `ctx.sessions` を提供し、参照、ソース・カウント、Session スコープ、安定した `SessionBinding` オブジェクトを所有し、グローバルな現在の Session を選択せずにカタログ状態を投影する。
- `SessionManager` は一覧のベースライン、ライブの一覧／制御更新、遅延 Session インスタンス、投影ストア、サブ agent カタログ、プルと後続の更新の間の競合順序を所有する。
- 各 `Session` は、`SessionEventLikeEntry` の値で表される 1 つの連続した論理イベント・ウィンドウ、ページング、フォロー、プロンプト／制御状態、アダプターが消費するオブザーバブルなスナップショットを所有する。

耐久イベントのパスは `follow()` を開く。その最初のフレームは現在のヘッダー、末尾ページ、カーソル、完全な投影ベースラインを含む。履歴レコードは明示的な `event` または `chunks` の判別子と、それに揃った内側の `event` を持ち、ジャーナルは包含的な論理シーケンス範囲をそれぞれ検証してから、Client がレコードごとの変換なしで `SessionEventLikeEntry` の値としてレコードを保持する。物理的な各世代はそのスナップショットから保持ウィンドウをアトミックに置き換え、その後の標準ライブ・イベントはシーケンス順に追加される。`page()` は古い履歴とギャップ修復のために予約されている。一時的な制御ストリームは、各世代を完全なベースラインで開始し、その後投影の更新を適用する。

### Workspace

[`api/workspace-controller`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/workspace-controller/README.md)は、Workspace の変更ポリシーと権威あるフォロー・フィードを Host 上に保持する。`ClientWorkspaceModel` はブラウザーの行、Workspace の順序、アーカイブ済みおよびピン留めされた Session id の配列、コマンドのエコー、ストリーム／単項の競合解決を所有する。各ストリーム世代は完全なベースラインで始まり、`upsert`、`remove`、`order`、`archived`、`pinned` の増分が続く。再接続は新しいベースラインでモデルを置き換える。`WorkspaceController` はそのモデルを `ctx.workspaces` として公開し、`ui-workspace` は `useWorkspaces` とナビゲーション・コールバックを提供する。サイドバーの `ArchivedFilter` は、一覧と検索でデフォルト非表示、表示、アーカイブのみの行を制御する。アーカイブされた行は順序スロットを保持し、灰色で描画され、行または検索結果の Unarchive アクションで復元されるまで開けない。復元は `workspace.unarchiveSession` を呼び、その完全なアーカイブ・セットは単項レスポンスと `archived` 増分を通じて Clients に届く。Session の表示順はブラウザー・ローカルに留まり、非表示のアーカイブを含む。ピン留めはその完全な順序の中で Session を移動させるが、ピン解除は以前の位置を復元しない。

この対はビジネス上の真実の第 2 のソースではない。Host コントローラーが耐久状態と変更結果を決め、Client モデルは最新の使用可能なローカル投影を維持し、レンダリングに有用な箇所でオブジェクト・アイデンティティを保ち、遅延したレスポンスと置き換えのベースラインがどのようにマージされるかをエンコードする。

## Conversation とプレゼンテーション

Web とデスクトップは[Coding Tools 設定](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-settings/README.md#use-this-package)を共有する。これは Session レコードを変えずに、診断ビュー、[利用可能なプリセットの選択肢](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-agent-preset/README.md)、変更ファイル・カード、組み込み HTML プレビュー・ポリシーを制御する。

`ui-session` は Session スコープのアダプターをインストールし、[remote-access](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/remote-access/README.md) Client サービスの上で `useSessions`、`useSessionStatus`、`useSessionRetainInfo`、`useSessionAccess`、`useSession`、`sessionId`、`useProjection`、および Session スコープの `useAccess` を公開する。`SessionProvider` は外側のバインディングを継承するか、明示的な `SessionReference` を束縛するので、並行するサブツリーは異なる Session をターゲットにできる。ドメイン・アダプターはモデル・オブジェクトに React フックを載せずに、さらなる標準ソースを追加する。

`ui-conversation` は各 `SessionBinding.eventSource` に 1 回束縛する。そのイベント・レジストリは耐久 Session イベントと Client 専用の `assistant/live-chunk` 更新を、安定したビジネス Context に相関させ、そのビュー・レジストリがターゲット・スナップショットを実体化する。Chat Assistant、Trajectory Assistant、Turn Tail はライブ・チャンクと、耐久決着に埋め込まれた圧縮ストリームの両方を解釈するので、再接続とページングされた履歴は、耐久トークン行なしで同じ Assistant 状態を再現する。`ui-chat` と `ui-trajectory` は別々の Definition とビルダーを登録する。同じイベント・ファミリーを解釈してよいが、互いの最終表示モデルをインポートしたり共有したりしない。シェルは登録されたビューを選択し、そのスナップショットを標準フックと Slots を通じて渡す。[Conversation](conversation.md)が Context のアイデンティティ、リプレイ、Location データ、ターゲット・ビルダー、キー付きレンダラーを定義する。

`ui-slots` は型付きレジストリとライフサイクル台帳を提供し、`ui-renderer` は素のオブザーバブルを `useSyncExternalStore` で束縛し、React コンテキストを所有し、ルート・ツリーをレンダリングする唯一のパッケージである。機能コンポーネントは導出された props を通じて、フレームワーク・フック、owner props、ストア・アクション、明示的なインジェクションを受け取る。[Web Client Slots](slots.md)がそれらの入力、拡張 API、現在の Slot 階層を一覧する。

## データ・パス

| パス | シーケンス |
|---|---|
| 耐久 Session の表示 | Host Session ログ → パックされた Remote の `follow`／`page` 履歴 → Client の `SessionEventLikeEntry` ウィンドウ → Conversation Contexts → ターゲット・スナップショット（`chat`、`trajectory`、または他の登録済みターゲット）→ Slot ビュー → React |
| 一時的な Session 制御 | Host 制御ベースライン → Remote スナップショット・ストリーム → `SessionManager` 投影ストア → Session と一覧のスナップショット → 標準フック → コンポーネント |
| バックグラウンド・ジョブ | Host ジョブ・レジストリ → `job.list` / `job.follow` → [`ClientJobs`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/job-controller/README.md) の名簿と出力ビュー → ジョブ一覧とパネル |
| Workspace 状態 | Host の Workspace ベースラインと増分 → `ClientWorkspaceModel` → `ctx.workspaces.list` → `useWorkspaces` → サイドバー、ヒーロー、ナビゲーション・エントリー |
| スコープ付きインタラクション | Host の Cordis ウォーターフォール → API Remotes の `$events` → Session Context 上の `ctx.remote.$on()` → 所有する UI パッケージ → 結果または `next()` |
| ユーザー・コマンド | コンポーネントのコールバック → 登録のインジェクト・フェイスまたは Slot の所有者 → `ctx.sessions`、`ctx.workspaces`、または生成されたスコープ付き Remote → Host Controller → 権威ある更新 → Client へのストリームまたはイベント投影 |

## 再接続

物理的な回復と論理的な回復は別である。Gateway の mux は物理 WebSocket を復元し、各 `RemoteStream` は Connection が使用可能な世代を公開したときに自身の論理ソースを再度開く。キャリアの失敗は再試行可能だが、ビジネス・エラー、不正な開始アイテム、プロトコル違反は、所有する論理ストリームにとって最終的である。

回復はデータのセマンティクスに従う。

- 耐久 Session ジャーナルは論理シーケンス範囲を検証し、各世代の開始スナップショットでウィンドウを置き換える。`page()` は古い履歴を供給し、後続の範囲ギャップを修復する。
- Session 制御と Workspace のストリームは、切断中に最後に公開された値を保持し、その後、新しい開始ベースラインでアトミックに置き換える。
- 通常の転送される通知は再生されない。ステートフルなドメインはベースライン、カーソル、または明示的なクエリを必要とする。スコープ付きウォーターフォールは自身の要求ライフタイムを保持する。

Client にモノリシックな `Runtime`、`HostFrame`、`events.mux`、`events.host`、万能の `resync()` API は存在しない。Connection は世代状態を公開し、Gateway は論理ストリームの監督を所有し、各 Client モデルは自身のデータに適した置き換えまたは再開のセマンティクスを定義する。

## パッケージの境界

機能プラグイン・パッケージは `import type` を通じて宣言を共有してよいが、別の機能プラグインの値をランタイム・インポートしたり再エクスポートしたりしない。パッケージ横断の振る舞いは注入された Cordis サービスを使い、パッケージ横断の UI は Slots を使う。ターゲット固有の Conversation Definition、投影ヘルパー、最終的なビュー・データは、Chat と Trajectory が意図的に並行するロジックを実装する場合でも、それぞれのターゲット・パッケージに留まる。

共有ランタイム値には、`client/store`、`ui-primitives`、ブラウザー安全なユーティリティ・パッケージのように、機能ライフサイクルを持たない狭い静的所有者が必要である。トランスポートと生成された API のアセンブリは、1 つのプロトコルを組み立てることが明示的な責務であるため、ランタイム寄与をインポートしてよい。機能パッケージはこの規則を回避するためだけに `dsh.client.external` を追加しない。

追加する拡張に応じて、4 つの詳細リファレンスを使う。

- パッケージの発見、ロード、共有モジュール・アイデンティティ、ブート順については[クライアント・モジュール](client-modules.md)。
- Host メソッド、生成された Remote の寄与、ストリーム、転送されるイベントについては [API Gateway](../api-gateway.md)。
- コンポーネント、フック、ストア、インジェクション、配置については [Web Client Slots](slots.md)。
- 耐久イベントの相関、ターゲット・スナップショット、Chat または Trajectory のビュー寄与については [Conversation](conversation.md)。
