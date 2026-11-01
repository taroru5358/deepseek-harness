# LiveShare

日本語 | [English](../../../docs/subsystems/liveshare.md) | [中文](../../../docs/subsystems/liveshare.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/liveshare.md](../../../docs/subsystems/liveshare.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

オプトインの LiveShare bundle（`@deepseek-ai/dsh-experimental-liveshare`）は、1 人のオーナーが別個の Liveshare サーバーを通じて 1 つの Session を外部ビューアーと共有できるようにする。ビューアーがオーナーのツールを受け取ることは決してない。オーナー host が各ビューアー・リクエストを自身で読み、そのビューアー自身の Peer として dispatch し、その `share-viewer` プリンシパルを Gateway がこのパッケージが登録する [remote access](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/remote-access/README.md) ポリシーを通じて決定し、共有 Session がビューアーの代わりに実行するすべてを、共有が許可したワークスペースに制約する。[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/liveshare/README.md)が設定、クライアント・フェイス、制限を所有し、[Owner note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/feature/2026-09-13-liveshare-protocol-and-owner-surfaces.md)がその背後のセキュリティ姿勢を所有する。

ソース: [`packages/experimental/liveshare/src/host/fork-enforcement.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/liveshare/src/host/fork-enforcement.ts)

<a id="enabling-and-visibility"></a>

## 有効化と可視性

このパッケージは dsh とともに出荷されるが、プロファイルが有効化するまでプラグイン・マネージャーのリストには現れない。プロファイルは、bundle が宣言する 2 つの行を自身の `cordis.patch.yml` に挿入する（プラグインが実行されている間も bundle は未選択のままになる）か、`dsh.profile.bundles` で bundle を選択する。[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/liveshare/README.md#default-visibility-and-enabling)が正確なエントリーと、デプロイが置き換えるプレースホルダーを所有する。

プラグイン・マネージャーは、bundle が宣言する行の少なくとも 1 つが実行されている間だけ、**Experimental plugins** の下にカードを列挙する。行を削除する、無効化する、または bundle の選択を解除すると、そのリストと **More** の両方から再び外れる。どちらの経路もタスクの途中ではなく、プロファイルの次の再合成時に適用される。HMR が有効ならプロファイルが読み直され、無効なら変更は次の起動を待つ。

## 共有モード

`ViewerSessionMode` には 2 つの階層があり、どちらも sandbox を指名する。`view` は共有 Session を読み、許可されたワークスペース内の読み取り専用 sandbox でそのループを駆動するので、ファイルを書き込むことは決してない。読み取りと検索のサーフェス（`read`、`read_image`、`glob`、`grep`、`present`）と、ローカル・ファイル効果を持たないツール（`todo_write`、`skill`、`web_search`、`web_fetch`）を使え、宣言されたすべてのパスは許可に対して検査される。`write` は境界付きの書き込み可能 sandbox で実行され、Host の書き込み上限の下で、境界付きの `edit` と `write` を追加してそのワークスペース内のファイルも変更する。fork は階層ではない。共有は階層とは独立した自身の `allowsFork` スイッチを持つ。添付ファイルのアップロードも階層ではない。共有がリーダーに提供するすべての Session は、階層や fork スイッチに関係なくファイルを受け入れる。撤回された `fork-readonly` 名は `view` として読み戻されない。それを名指す永続オーナー・レコードは、オーナー状態のロード時に拒否される。何も `write` を推論せず、host がモードを確立できない共有は読み取り可能なままだが、駆動されることは決してない。

## fork とその上限

fork はビューアーが所有する新しい Session を作成し、ソースを変更しない。fork の認可は、共有、その fork スイッチ、有効期限、ビューアーの受入、ソース・アイデンティティ、要求された境界を検査する。`READ_ONLY_FORK_CEILING` は view サーフェス — `read`、`read_image`、`glob`、`grep`、`present`、`todo_write`、`skill`、`web_search`、`web_fetch` — を許可し、`WRITE_FORK_CEILING` は Host の境界付き `edit` と `write` を追加する。パスを持つ各ツールは、宣言されたすべてのパスを許可されたルートに対して解決し、正規形で委譲する閉じ込められた置き換えを受け取る。`bash`、`pwsh`、`cua`、`plugin_manager`、`schedule_*` ツール、およびサーフェスが列挙しないすべての名前は拒否される。

各子は、それを認可した共有、Liveshare サーバーが主張した信頼済みビューアー、開始した境界とともに記録されるので、ビューアーのリストには自身の子だけが表示され、削除された共有はそれらを停止する。子は、そのガードが自身のスコープにインストールされた後にだけ駆動される。`ForkRegistry` はガードがインストールされていない子へのすべてのプロンプトを拒否し（`fork-ceiling-uninstalled`）、インストールは host ごとではなく子ごとに検証される。スコープが閉じ込められた置き換えまたは上限を拒否する子は、デプロイ自身のツールで公開されるのではなく拒否される。

コールド・ソースと子のアクティベーションは、Remote の `session/prepare` エンドポイントではなく、Host の通常の Agent ルックアップを使う。直列の `agent/created` リスナーが、キューされた作業が解放される前にスコープ付きの強制をインストールする。既にライブの Agent は自身のインストール検査を必要とする。`forkLineage` は LiveShare 所有のビューアーごとの UI projection であり、Session サマリー・フィールドでも、永続ログ・イベントでも、権限付与でもない。

## リトライ、復旧、失効

1 つの論理 fork リクエストが複数回送られることがある。そのアイデンティティは、信頼済みビューアー、共有、ソース、呼び出し元の操作 id を束縛する。境界はそのアイデンティティに折り込まれるのではなく、それと比較されるので、リトライは既に生成した子を繰り返し、別の境界を名指すアイデンティティは以前の子を黙って受け取るのではなく競合し、予約は子が存在する前に永続化される。host はオーナーの fork 応答が返した子を記録し、記録済み操作のリトライはその応答をリプレイする。オーナーの `SessionForkRequest` が子 id を発行するので、子の作成とその記録の間に停止した host は、リトライ時にその子を名指せず 2 つ目の子を作成する。この窓を閉じるには、呼び出し元が指定した子アイデンティティを受け入れる Session Controller リクエストが必要である。

子のツール・ガードは、共有の現在のモード、有効期限、ビューアーの受入、失効レコードを再検査する。ビューアーの撤回は、影響を受ける子の作業と保留中の呼び出しをキャンセルする。`write` から `view` へのダウングレードは、新しいツールを付与せずに書き込みガードを通じて以降の書き込みを拒否し、実行中の turn の即時キャンセルを約束しない。共有を削除すると記録済みの子が失効し、Session の実行中の turn が停止する。ソースの強制層は、Session 上の共有が残っていなくなると解放され、Agent の破棄でも解放される。失効した子はログを保持し、ソースのオーナー可視履歴が書き換えられることは決してない。

## 共有 Session の制約

すべての共有は、そのビューアーが通常のプロンプト経路を通じて*ソース* Session を駆動できるようにし、どちらの階層の共有もアクティブな間は境界付き sandbox をインストールする。`installSourceSessionAccess` は、その Session 自身のスコープにデプロイが出荷するすべての view サーフェス・ツールの閉じ込められた置き換えを登録し、Session 上のいずれかの共有が書き込みを許すときは常に、境界付きの `edit` と `write` の置き換えを登録する。層は Session にマウントされるが、共有ビューアーが開始した turn だけを閉じ込める。`agent/pre-step` は、turn の受入済み入力が共有ビューアーを名指すかどうかを Host のメモリーに記録し、ガードと置き換えはその turn のすべての実行についてその回答を読む。ビューアーが開始した turn はそれらの置き換えだけを実行し、オーナーが開始した turn は `bash` とその無制限のファイルシステム・ツールを含む Session 自身のツールを保つ。回答は turn が受け入れられたときに決まり、Session ログには決して書き込まれない。再起動時には、次に受け入れられた step から再計算される。ビューアーの turn 内でのオーナーのステアリング入力は、その turn とともに閉じ込められたままになる。Session 上のすべての共有を停止すると層が破棄され、Session 自身のツールが復元される。最後の write 共有を `view` に狭めると、読み取り専用層が再インストールされる。

インストールは、ソース Session が実際に閉じ込められているかどうかを報告する。`read` ツールを持たないデプロイ、またはその置き換えの登録を拒否するスコープは、ガードをそのままにして拒否を報告する。自身の置き換えが失敗したツールは、サーフェスの残りが実行されている間も拒否されたままで、リーダーは `source-write-unavailable` でプロンプトを拒否する。上限をインストールできなかったために、共有 Session のビューアー開始 turn がオーナー自身のツールに対して駆動されることはない。

各操作は、Session を駆動可能に保つ共有の Relay 許可を再読み取りし、各読み取りはそれが認可するツール呼び出しと同期的であるので、それを認めた許可を超えてキューされた入力がその陳腐化した受入に作用することはできない。層は Session ごとである。1 つの Session 上の複数の共有は 1 つのインストールを共有し、狭めるか終了する共有は層が許すものを変える。アクティブな共有が書き込めなくなると、その境界付き書き込みは拒否され、共有が 1 つも残らなくなると層は解放される。Host は実行中の turn を、それを認めたビューアーに帰属させないので、1 つの共有の許可を終了すると、そのビューアーの作業だけを特定するのではなく Session の実行中の turn が停止する。メモリー内のビューアー開始マーカーは、その turn がどのツール・サーフェスを実行するかだけを決める。

閉じ込められたファイル・ツールは、自身の引数が宣言するすべてのパスを解決し、許可されたルートが各結果を含むことを確認し、ターゲットの正規パスで委譲する。パスのない `glob` や `grep` は Session の作業ディレクトリを検索し、置き換えがそれを名指しのパスと同様に提供して検査する。`present` 呼び出しは列挙するすべてのファイルを検査する。ローカル・ファイル効果を持たないツールは変更されずに委譲される。境界付き編集は同じ封じ込め検査を実行し、`fs/edit-intent` を使い、`editText` が返すバージョンを公開する。境界付き書き込みは `fs/write-intent` を使い、`writeText` が返すバージョンを公開する。非同期の intent 待機の後、Host は現在の Session sandbox ポリシーを解決し、共有の権限を再検査する。そのポリシーを `signal` の後の 5 番目の `editText` または `writeText` 引数として渡すので、sandbox バックエンドはそれによって変更をフェンスする。封じ込めはパスで検査されるので、許可内のディレクトリーが検査と開くの間のどこかでローカル・プロセスによってシンボリックリンクに置き換えられると、操作をリダイレクトできる。操作をディレクトリー・ハンドルに固定することは別の変更に延期されている。

ビューアーのソース・プロンプトとソース停止はどちらも、`share-viewer` ポリシーの source-write ロールの下で標準の Gateway dispatch を通る。ポリシーは共有の階層が確立され、その Session が閉じ込められている間はそれらを認め、その後オーナー自身の Session コントローラーが Session の実行中の turn を停止する。キャンセルはビューアーごとの比較を行わないので、リーダーによる共有 Session の停止はオーナーが開始した turn も終了させる。[Session Controller](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/session-controller/README.md)は、`prompt` の内側で配送の直前に、`cancel` と同じ同期ステップで `remoteAccess.allows()` を再度読むので、Gateway がその呼び出しを認めた後にアクセスが終了したビューアーは何も配送せず、何もキャンセルしない。キューされた外部ソース入力は `agent/pre-step` で Liveshare サーバーの権限をリフレッシュし、権限がない場合はモデル・リクエストの前にその step を拒否する。

<a id="viewer-states"></a>

## ビューアー状態

拒否で終わるビューアーの読み取りは、すべての原因に対して 1 つの失敗を返すのではなく、自身の条件を名指す。Liveshare サーバーは、リンクの欠落、オーナーが停止した共有、期限切れの共有、ルートを持たないオーナーのマシン、期限切れのサインイン、まだ要求していないリーダー、待機中のリクエスト、オーナーが拒否したリクエスト、オーナーが受け取らなかった通知に、それぞれ固有のステータスとコードで答える。Liveshare サーバーに到達できないページは、そのことを自身で述べる。`@deepseek-ai/dsh-experimental-liveshare-protocol` の `VIEWER_STATE_ANSWERS` が語彙全体であり、[共有プロトコル](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/liveshare-protocol/README.md)が各条件の運ぶ回答を所有する。どの回答もビューアーが読めるものを広げない。オフラインのオーナー、停止した共有、拒否されたリクエストはすべて拒否する。

ビューアーのページがレンダリングするワークスペース・フィードは、そのセッション・リストと同じポリシーでフィルターされる。ワークスペースは、共有 Session またはそのビューアーがそこから作成したブランチを保持している間だけ配信され、そのメンバーシップと手動の順序はそれらだけを保つ。それらを失ったワークスペースは、そのワークスペースまたはワークスペース順序が次に変わったときにフィードを離れ、その後ストリームはリーダーのベースラインを置き換える。アクセスの変更だけではフレームを送らない。したがって、リーダーは許可された 1 つの Session の周りのオーナーのワークスペース名とグループ分けを見るだけで、オーナーの残りのワークスペースを見ることはない。

ビューアー・ストリームは、ポーリング、置き換え、キャンセルをまたいで開始アクターを束縛する。Session 通知、制御ベースライン、ワークスペース・メンバーシップは、ソースとそのアクターの子だけを公開する。Gateway はリーダーに、ポリシーがそれらの Session で許可する `api-session/*` 通知だけを転送し、Session と Workspace のコントローラーは同じポリシーを通じてリストとストリームをフィルターする。対話型の承認と質問は Host に留まる。拒否された読み取りは上流の購読を開かない。有効期限と失効は影響を受ける保留中の呼び出しとストリームをキャンセルし、ポーリングされないストリームは 120 秒後に回収される。Worker IPC の応答とキャンセルは、リクエストのトークン、共有 id、Liveshare サーバー id、元のソケットまたはワーカー世代に一致しなければならず、遅れて届いた回答が置き換えリクエストを決着させることはできない。

Liveshare サーバーの任意の `share-access-state` ケイパビリティは、接続時とコミットされた変更時に、別の Liveshare サーバー・インスタンス上の変更も含めて、完全でバージョン付きの権限スナップショットを公開する。認証済みの `GET /__liveshare/requests` は同じフィールドを返す。Host は接続損失時にキャッシュされた権限をクリアし、世代内の陳腐化したリビジョンを拒否し、非同期リフレッシュを世代変更に対してフェンスする。完全な権限がない場合は受入を拒否する。これは即時の分散停止セマンティクスを確立するものではない。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。dispatch モードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxliveshareshares--liveshareshares"></a>

### `ctx.liveshareShares` — `LiveshareShares`

共有管理操作に対するオーナー専用の Remote サーフェス。

Gateway は remote access を通じて各呼び出しを決定するので、`share-viewer` Peer はメソッドが実行される前に拒否される。オーナー Peer は、各共有管理の副作用を所有する唯一の実装に到達する。`create` はその Session を単一のリクエスト・オブジェクトの `sessionId` として取るので、Session リソース種別の既定リゾルバーが、ここでリーダーを宣言することなくその呼び出しの Session を指名する。

```ts cordis-catalog
/**
 * Publish one Session and store the share.
 * @param request - Session, title, access, lifetime, and both switches the owner chose.
 * @returns The created share's link and stored values.
 */
@Remote('create') create(request: CreateShareRequest): Promise<CreatedShare>

/**
 * Refresh titles and Liveshare server authority, then list this host's shares.
 * @returns Every share this host publishes.
 */
@Remote('list') list(): Promise<OwnerShareList>

/**
 * Stop one share locally, revoke its forks, and delete its row on the Liveshare server when reachable.
 * @param request - Share to stop.
 * @returns Whether the row on the Liveshare server still needs a retry.
 */
@Remote('stop') stop(request: StopShareRequest): Promise<StopShareResult>

/**
 * Change one share's tier, fork switch, or both.
 * @param request - Share and the switches to change.
 * @returns The permissions the share now carries.
 */
@Remote('updatePermissions') updatePermissions(request: UpdateSharePermissionsRequest): Promise<SharePermissions>

/**
 * Replace one share's exposed plugin endpoints.
 * @param request - Share and the selection to store.
 * @returns The capability selection the share now carries.
 */
@Remote('setCapabilities') setCapabilities(request: SetShareCapabilitiesRequest): Promise<ShareCapabilities>

/**
 * Read the endpoints plugins currently offer.
 * @returns The live offer list.
 */
@Remote('capabilities') capabilities(): ShareCapabilityOffers

/**
 * Read the Liveshare server address the owner saved.
 * @returns The current Liveshare server state, with both address fields absent while none is saved.
 */
@Remote('relay') relay(): OwnerRelayState

/**
 * Choose the Liveshare server address later shares publish to.
 * @param request - Normalized address or null to clear the saved one.
 * @returns The Liveshare server state after the change.
 */
@Remote('setRelay') setRelay(request: SetRelayRequest): Promise<OwnerRelayState>

/**
 * Report this host's owner identity for one Liveshare server address.
 * @param request - Address to report, or the effective one when absent.
 * @returns The owner identity status, without the credential.
 */
@Remote('identity') identity(request: OwnerIdentityRequest): Promise<OwnerIdentityStatus>

/**
 * Read the Liveshare server's durable audit trail for one share.
 * @param request - Share whose audit is read.
 * @returns The audit events, or a structured failure when the Liveshare server cannot answer.
 */
@Remote('audit') audit(request: ShareAuditRequest): Promise<ShareAudit>

/**
 * Check that the effective Liveshare server answers at all.
 * @returns The anonymous connectivity result.
 */
@Remote('testRelay') testRelay(): Promise<RelayConnectivityView>

/**
 * Register one throwaway share to check whether this host may publish.
 * @param request - Session the throwaway registration names.
 * @returns The Liveshare server's decision about this host's credential.
 */
@Remote('testPublish') testPublish(request: TestPublishRequest): Promise<RelayPublishView>
```

ソース: [`packages/experimental/liveshare/src/host/share-remote.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/liveshare/src/host/share-remote.ts)
<!-- END GENERATED cordis-surface -->
