# DeepSeek Harness アーキテクチャ

日本語 | [English](../../docs/architecture.md) | [中文](../../docs/architecture.zh.md)

> これは非公式の日本語訳です。英語版（[docs/architecture.md](../../docs/architecture.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

`packages/` 以下を変更する前に、この文書を読んでください。Cordis の知識を前提としています。Cordis を知らない場合は、先に [primer](../../docs/cordis-primer.md) または [tutorial](../../docs/cordis-tutorial/index.md) を読んでください。

## Cordis

[Cordis](../../docs/cordis-primer.md) は dsh の基盤フレームワークです。プラグインは、共有コンテキストにサービス、型付きイベント、可逆な効果（reversible effects）を提供します。モデル・アダプター、ツール・レジストリ、セッション・ログ、agent loop 自体を含め、製品のあらゆる部分がプラグインであり、それぞれを設定から差し替えられます。

特権的なコアを patch するのではなく、他のプラグインの隣にプラグインをマウントすることで dsh を拡張します。登録は効果であり、そのプラグインがアンロードされると巻き戻されます。

## プロファイルとバンドル

実行中の `dsh` は、起動時に順序付けられたレイヤーから構成されるプラグイン・ツリーです。

**プロファイル**は、Harness home に保存された名前付きの構成です。積み重ねるバンドルを列挙し、ツリー外のプラグインのインストールを保持し、ユーザー自身の `cordis.patch.yml` を保持します。`web`、`headless`、`sdk`、`sdk-minimal`、`acp` がテンプレートとして同梱されています。

**バンドル**は、Cordis の設定行と、それがマウントするコードの配布形式です。バンドルが挿入したものは、上位のレイヤーから引き続き patch できます。

それぞれは自分の `package.json` の `dsh` フィールドで自身を宣言します。`dsh.profile` はプロファイルのバンドルを列挙し、`dsh.bundle` はバンドルの patch ファイルを指します。

[`dsh-base`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/base/README.md) は、`web`、`headless`、`sdk`、`acp` プロファイルの共有された最初のレイヤーです。モデル・アダプター、ツール、永続化、sandbox と承認ポリシー、設定、認証情報、テレメトリーを含みます。[`dsh-web-app`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/web-app/README.md) はブラウザ・アプリケーションを追加し、[`dsh-headless`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/headless/README.md) はサーバーを持たないワンショット・ランナーを追加し、[`dsh-sdk-app`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/sdk-app/README.md) は SDK JSON-RPC サーバーを追加し、[`dsh-acp-app`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/acp-app/README.md) は自動化専用の ACP サーバーを追加します。[`dsh-sdk-minimal`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/sdk-minimal/README.md) は意図的な例外です。1つのバンドルが完全に明示的な SDK ツリーを所有し、`dsh-base` を適用しません。

レイヤーは空のエントリー・リストに適用されます。プロファイルの順序付きバンドル、その `cordis.patch.yml`、home の patch、最後に `--patch` のオーバーレイ、という順序です。patch は id で行を対象にし、その設定全体を置き換えるか、新しい行を挿入します。`preset` patch は、同じレイヤー優先順位で、その宣言の `config.plugins` 内にこれらの操作を適用します。

HMR は YAML で制御します。base は設定のみの `dsh-hmr` を有効にし、headless、SDK、ACP では無効にし、`sdk-minimal` では省略します。プロファイルの patch はこれらの既定値を上書きします。HMR はファイル監視とリロードを調整し、ランチャーはプロファイル・データと準備完了状態を提供します。

base には、Web とエージェント向けの [Plugin Manager](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-manager/README.md) が含まれます。

自分のマシンが起動するツリーを確認するには、次を実行します。

```sh
dsh --profile web --dump-config
```

出力されたどの行も、自分の patch で置き換えられます。

構成の仕組みは [app-boot](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/app-boot/README.md#profiles) に、設定フィールドは生成された [config catalog](../../docs/config-catalog.md) にあります。

## アプリケーションの起動

対応する Node アプリケーションは、名前付きの `dsh` プロファイルを通じて起動します。同梱されるプロファイルは `web`、`headless`、`sdk`、`sdk-minimal`、`acp` で、`dsh --profile <name>` または `dsh <name>` で選択します。`plugin` は管理コマンドの名前です。その名前のプロファイルを使うには `--profile plugin` が必要です。TypeScript SDK は同じバージョンの `dsh` 依存関係を解決して `sdk` を選択します。カスタム・プラグイン構成は、別の実行ファイルやインラインのアプリケーション・ツリーではなく、プロファイルと順序付き patch ファイルのままです。`sdk-minimal` は、同じランチャーの背後にあるリポジトリ所有のスタンドアロン・バンドルであり、呼び出し元が提供する Cordis ツリーではありません。

同梱された CLI（vendored CLI）、ビルド専用・テスト専用の実行ファイル、インプロセスでの直接プラグイン・マウント、プライベートなブラウザ WebWorker プレビューは、Harness のアプリケーション・ランチャーではありません。[`verify-application-entrypoints`](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/verify-application-entrypoints.ts) は、各パッケージの bin、実行可能ソース、ルート・デモ、ルートの `start:web` / `dev:web` スクリプトを明示的なクラスに保ち、`dsh` を迂回する Node アプリケーション・パスを拒否します。

Python SDK も同じアプリケーション・アーキテクチャに従います。そのランタイム wheel は、通常の `dsh` CLI を `deepseek-harness-sdk-runtime-<platform>-<arch>` としてパッケージし、クライアントは既定で明示的な Harness home を指定して `dsh --profile sdk` を起動します。最小の例では、同梱の `sdk-minimal` プロファイルを選択します。Python は完全な Cordis ツリーではなく、プロファイル選択と順序付き patch ファイルを公開します。永続的な外部プラグインは `dsh plugin` でインストールします。削除されたプライベートな直接設定キャリアには、互換 bin もフォールバック・パーサーもありません。

## デスクトップ・アプリケーション

[Electron デスクトップ・アプリケーション](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/desktop/README.md)は、署名済みリソースに dsh の本番ランタイムをそのまま同梱し、予約された `$DSH_HOME/profiles/desktop` を所有します。共有ヘルパーはプロファイル・ファイルを初期化し、バンドルを調整し、pnpm 所有のパッケージを置き換えずにインストールとバンドルの依存関係を解決します。デスクトップと npm CLI は製品データを共有しますが、パッケージ、アクティベーション、ロックファイルは分離しています。デスクトップに同梱された CLI が、初期化されたプラグインを管理します。

Electron は、Electron の Node モードでプライベートな Desktop Host を起動します。Host は共有 CLI のプロファイル・ランナーと完全な Web アプリケーションを呼び出します。ウィンドウはパッケージ済みの Web アセットを即座に読み込み、同じドキュメント内でクライアント・プラグインを有効化する前に boot injection を待ちます。Web は RPC とストリームを所有し、デスクトップ・キャリアがローカル・ページを認証済み Host に接続します。Node IPC は boot injection、準備完了、致命的エラー、シャットダウンを運びます。デスクトップは既定で OS が割り当てたポートで待ち受け、プロファイル設定で上書きできます。シェル所有の UI は、通常のユーザー設定とプロファイル設定で、同梱 pnpm を通じてプラグイン・トランザクションを実行します。

## コア・パッケージ

| パッケージ | 所有するもの | `ctx` キー |
|---|---|---|
| [`core/session`](../../docs/subsystems/session.md) | 追記専用の `SessionEvent` ログとインメモリ・ストア | `ctx.sessions` |
| [`core/system-prompt`](../../docs/subsystems/system-prompt.md) | プロンプト・セクションとツール・スキーマの組み立て | `ctx.systemPrompt` |
| [`core/tools`](../../docs/subsystems/tools.md) | スコープ付きツール・レジストリとガード付き実行パイプライン | `ctx.tools` |
| [`core/agent`](../../docs/subsystems/core.md) | `Agent` インターフェース、ライブ・レジストリ、`agent/*` イベント | `ctx.agents` |
| [`core/agent-loop`](../../docs/subsystems/core.md) | そのインターフェースを実装する既定のドライバー | `ctx.agentLoop` |
| [`core/scope`](../../docs/subsystems/scope.md) | エージェント単位のスコープ付き登録プリミティブ | ライブラリ、キーなし |
| [`llm/llm`](../../docs/subsystems/llm-streaming.md) | メッセージとストリームの語彙、アダプター・シーム | `ctx.llm` |

## イベント

イベントは拡張ポイントであり、ほとんどの変更では、どのドメインを選ぶかが最初の判断になります。

- **セッション・イベント**は、ログに追記され、`session/event` でブロードキャストされる永続的な事実です。リロード後も残す必要がある事実には、これを使います。
- **エージェント・イベント**（`agent/*`）は、ライブな `Agent` を運びます。inbox、step、status、request、validation、continuation などです。実行中の作業を観察・傍受するには、これを使います。
- **ケイパビリティ・イベント**は、ループをインポートせずに、ポリシーとアダプターをシーム（`fs/*`、`tools/*`、`telemetry/*`）に接続します。

AgentLoop は、ロールバックを伴って Agent を初期化し、キューされた作業の前に直列な `agent/created` を待ちます。アンロード時は、Agent の teardown が完了するまでリソースを保持します。順序は [agent-loop](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md#understand-the-implementation) が定義します。

[イベント・マップ](../../docs/event-producer-consumer.md)には、各イベントの生産者と消費者が一覧されています。

## ターン・フロー

**ステップ**は、1回のモデル・リクエストと、その応答が呼び出すツール実行です。**ターン**は0個以上のステップです。ターンは、最初の入力が claim される前に開き、未処理のものがなくなった時点で閉じます。

```text
turn/start
  claim next-step input plus one queued message
  snapshot Session model selection; assemble prompt sections + step-admitted tool schemas
  -> agent/pre-step; project runtime context at fallback
                                     reject | enter(messages, startsRequestSeries?)
     reject, or a first enter rewritten empty -> close the turn with no step
     step/start
     agent/request -> prepareCall (cancellation commits neither system nor users)
     reconcile system/message using the prepared call capability
     append entered messages as user/message; reconcile retained runtime context
     log request/header and request/context as needed
     derive and freeze model history from the log
     stream the bound prepared call -> llm/stream -> agent/assistant-stream start
       agent/assistant-stream chunk*
       assistant/message | assistant/attempt -> agent/assistant-stream end
     tool/call* -> tools/pre-execute -> tools/execute -> tools/post-execute -> tool/result*
     step/end
     tools owe another request, or next-step input arrived -> claim -> next step
  -> agent/turn-stopping
turn/end
```

`turn/*`、`step/*`、`system/message`、`user/message`、`assistant/message`、`assistant/attempt`、`tool/*` は永続的なセッション・イベントです。それ以外は、3つのドメインにまたがるライブな拡張ポイントです。`agent/assistant-stream` は、プロセス・ローカルな start、一時的な chunk、end フレームを発行します。ループは、完全なコンパクト・ストリームを1つのメッセージまたはログ専用の attempt として、コミット済みの end フレームの前にコミットします。Web の Session-follow アダプターは、ライブ・イベントの唯一のリモート・コンシューマーです。`agent/pre-step`、`agent/request`、`llm/stream`、3つの `tools/*` イベントはウォーターフォールであり、リスナーは委譲のために `next()` を呼ぶ必要があります。`agent/turn-stopping` は直列で、`next()` を持ちません。

ファクトリーのセットアップは、SessionStore への登録と `session/created` に続き、`agent/created` の前に行われます。終了イベントは削除の前にコミットされます。

1つの inbox がドライバーに供給します。注入されたコンテキストは、起床メッセージを待ちます。AgentLoop の永続的な `inbox` プロジェクションは、ライブな Agent なしで保留入力を公開します。

組み立ての前に、ループは `AssembleContext.modelSelection` と `agent/pre-step.modelSelection` でセッションのモデル選択をスナップショットします。リトライはそれを保持し、省略された reasoning effort はアダプターの既定値を使います。`agent/pre-step` のリスナーは、claim されたメッセージを受け入れるか、書き換えるか、拒否します。拒否された場合、または最初の claim が空の場合、ターンはステップなしで永続的に閉じます。enter の判断は `startsRequestSeries` を設定できます。ループは新しい `request/header` を記録します（理由は `series`、またはエンベロープも変わった場合は `startsSeries: true` を伴う `change`）。ラップするリスナーは、その宣言を `{ ...decision, messages }` で保持します。組み立てと `step/start` の後、非推奨の `agent/request` フックは、プロバイダー、モデル、effort の選択を変えずに制御を調整します。`prepareCall()` は、システム・プロンプトと受け入れ済みユーザーがコミットされる前にアダプターをバインドします。どちらかの非同期フェーズでのキャンセルは、どちらもコミットしません。準備済み呼び出しのケイパビリティは、先行する `request/context` ではなく、プロンプトの受付を支配します。各試行は、登録済みのランタイム事実を更新し、バインドされたプロンプトを調整し、ユーザーを一度だけ受け入れ、削除されたランタイム・コンテキストを復元し、必要に応じて header/context を記録し、バインドされた呼び出しをストリームする前にリクエストを凍結します。リトライは、組み立てや `agent/pre-step` を繰り返しません。添付後の surface 置換と画像オフロードの判断は、最初に再開された pre-step 中も含めて、新しいリクエスト・シリーズを開始します。変更のない resume はシリーズを継続します。システム・プロンプトは `system/message` 履歴として運ばれます。空テキストはそれをクリアし、対応可能なルートは更新を追加し、それ以外のルートは最初のシステム・ノードで統合します。プロンプトの受付と置換の詳細は [Agent-loop](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md#understand-the-implementation) が所有します。

ループは、ライブなキャンセルを伴う不変のリクエストを送信し、凍結された同一性には凍結証拠を再利用します。構築とキャンセル原因は [agent-loop](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md) が所有します。

失敗したステップは、[欠落したツール結果を記録します](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md#understand-the-implementation)。

詳細は、[シーケンス図](agent-lifecycle.md)、[ツール・パイプライン](tool-execution-pipeline.md)、[キャンセルとエラー回復](../../docs/subsystems/core.md#the-agent-handle)を参照してください。

[`ctx.tools.execute()`](../../docs/subsystems/tools.md) は、生のバッチと並行性の上限を受け取ります。

## セッション・ログ

セッション・ログは、モデルが見るコンテキストの源泉です。`deriveMessages()` は、そこからモデル履歴をプロジェクションします。各 `assistant/message` は、その組み立てられた内容を生成した正確なコンパクトな timed stream を埋め込みます。`assistant/attempt` は、モデル履歴を追加せずに、決着した失敗、リトライ、キャンセル、ストリーム・エラーの試行を保持します。fork、resume、transcript、telemetry、永続化はすべて、これらの永続的な決着から派生します。ライブ UI の増分性は `agent/assistant-stream` から来ます。決着前のハードなプロセス損失は、永続的な attempt ストリームを残しません（[decision](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-01-v2-embedded-assistant-streams.md)）。

セッション・コンシューマーは、現在の論理フォーマットだけを知っています。ヘッダーだけの `stat` と `list` は、各セッション・ディレクトリを再スキャンし、数値上最も高い正規の世代を選択し、サポートされる履歴ヘッダーを、イベントをロードせず、後継を公開せずに変換します。保存済みセッションの `open` は、同じ世代を選択するか、未来のバージョンを拒否するか、静的な隣接マイグレーション・チェーンを一度デコードして合成してから、検証済みの現在の論理イベントを返します。読み取り open は、その後継を公開せずにインメモリ結果を使います。書き込み open は、まずエンコードして検証し、変更されていないソースの隣に最終的なバージョン名付き後継を排他的に公開します。封印されていない中断テールの通常の修復は、ハンドル・コンシューマーの責任のままです。マイグレーションは、欠落した中断 `turn/end` を、後の `turn/start` ですでに封印された境界付きリリース済み再起動に対してのみ挿入します。JSONL v0 は `session.jsonl[.zstd]` を使い、v1 以降は小文字の `session.vN.jsonl[.zstd]` を使います。コミット済みの世代パスは、名前変更、置換、削除されません。JSONL プロバイダーは、物理フレーミング、圧縮、世代選択、排他的公開を所有し、各隣接マイグレーション・パッケージはちょうど1つの `vN -> vN+1` ステップを所有します（[decision](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-31-released-session-format-migrations.md)）。

**モデルに見えるものは、ログに残っていなければならない。** すべてのモデル・リクエストは、ログから再構築できなければなりません。新しいモデル可視入力には、セッション・イベントが必要です。既存のメッセージ内容を変更するプラグインは、[純粋なメッセージ・プロジェクション](../../docs/subsystems/session.md#plugin-owned-message-projections)を登録します。デタッチされたリーダーは、同じ定義を明示的に提供します。ツールの変更はケイパビリティに依存しません。[Session tool history](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/README.md) がプロバイダー宣言を提供します。

**プロジェクション・シーム。** `dsh-session-projection` は `ctx.sessionProjections` を所有します。登録されたユニットはコミット済みイベントを畳み込み、キャリアは `snapshot()` を通じて選択されたクライアント・ビューを読みます。ドメイン・サービスは登録を待ち、プライベート・ハンドルを保持し、ビジネス・メソッドを公開します（[登録ライフサイクル](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session/session-projection/README.md)）。`AgentRegistry` は `turnBoundaryOf(session)` を通じてターンとステップの状態を公開します。`AgentLoop` は `modelSelection` と inbox 状態を所有します（[ループ・ライフサイクル](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md)）。

[作業ディレクトリ](../../docs/subsystems/working-directory.md)は、元のプロジェクト同一性、sandbox の書き込みルート、既存のプロセス・ディレクトリを変えずに、ユーザー・コンテキストと実行パスを提供します。

## ケイパビリティ・シーム

**シーム**とは、交換可能なケイパビリティであり、3つの役割を持ちます。インターフェースを宣言する**サービス定義**、それを実装する**サービス・プロバイダー**、それを使う**コンシューマー**です。コンシューマーは通常、モデル向けツールです。1つのパッケージが複数の役割を兼ねることはありますが、1つの役割だけではシームではありません。ケイパビリティを追加するとは、3つすべてを設計することです（[ケイパビリティ・グラフ](../../docs/capability-seams.md)）。

シームがあるため、プロバイダーを1つ差し替えるだけで製品全体が変わります。ファイルシステムとサブプロセスのプロバイダーは1つの実行世界を共有するため、それらをリモート sandbox に向けると、Bash、PTY、LSP も一緒に移動し、プロバイダーのフォークは不要です。[サブエージェント・プロバイダー](../../docs/subsystems/subagent.md)も同様に、1つのインターフェースの背後で、新しい子エージェントから別製品への委譲ターンまで幅広く変わります。

[実験的な Agent Teams](../../docs/subsystems/agent-team.md) は、オプトインの `ctx.agentTeams` コーディネーションを公開します。永続的なロスターとタスク状態、継続可能なサブエージェントに対する直接の inbox メッセージングを持ちます。

## 新しい振る舞いを置く場所

新しい振る舞いは、文書化された拡張ポイントに接続します。ループ自体を変更する場合は、このマップを更新します。

| 目的 | メカニズム |
|---|---|
| モデル・プロバイダーを追加する | `ctx.llm` にアダプターを登録する |
| モデル向けケイパビリティを追加する | `ctx.tools` に登録する。スキーマはプロンプト組み立てに参加する |
| 1つのセッションに異なるケイパビリティ・セットを与える | エージェント・プリセットを構成する。そこにあるサービス行には `isolate` レルムが必要 |
| シェル実行を追加する | `ctx.shell` バックエンドを登録する。ローカルのものは `ctx.subprocess` を通じて spawn する |
| 永続的なターミナル実行を追加する | `ctx.terminals` バックエンドと `dsh-tool-terminal` を登録する |
| ヒューマン・コマンドを追加する | `ctx.commands` に登録する。モデル・ターンなしでディスパッチする |
| バックグラウンド・ジョブを管理する | `ctx.jobs` に登録する。`job_*` ツールがジョブを読み取り・停止する |
| 外部 webhook からセッションを開始する | `ctx.webhookRuntime` に信頼済みルールを登録し、プロバイダー・アダプターをマウントする |
| ファイルシステム・アクセスやポリシーを追加する | `ctx.fs` プロバイダーを登録するか、`fs/*` イベントを購読する |
| spawn されたプロセスを閉じ込める | `ctx.sandbox` バックエンドを使う。コンシューマーは spawn 前に argv をラップする |
| リクエスト、ツール、ターンを傍受する | その `agent/*` または `tools/*` イベントを使う。`agent/turn-stopping` はターンを停止する |
| モデル向けコンテキストを追加する | `agent.inject()` を呼ぶ。次の受付リクエストに入る |
| UI やエディタ連携を追加する | `ctx.agents` を駆動し、`session/event` から描画する |
| Web Client Chat ノードを追加する | `ConversationNodeDefinition` とキー付きレンダラーを登録する |
| 永続的なセッション状態を追加する | `SessionEventMap` を拡張する。ログから描画・リプレイする |
| セッション・タイトルを生成する | 唯一の `ctx.sessionTitle` プロバイダーを登録する |
| 同一セッションの目的を管理する | `ctx.goals` を使う。`agent/*` を通じて継続する |
| ターン境界で fork する | `ctx.agents.create({ sessionId, seed, inheritedEventCount, meta: { isSeeded: true } })` — agent-loop が公開したセッションだけが永続化される |
| セッションを新しいバックエンドに保存する | 共有ハンドル・スキャフォールディング上に `SessionPersistence`（`create`/`open`/`stat`/`list`/`export`）を実装する |
| 登録を1つのエージェントにスコープする | そのエージェントの `agent.ctx` を使う |

[拡張クックブック](../../docs/cookbook/extension-cookbook.md)は、機能をケイパビリティに対応付け、[パッケージ](../../docs/cookbook/adding-a-package.md)、[ツール](../../docs/cookbook/adding-a-tool.md)、[LLM アダプター](../../docs/cookbook/adding-an-llm-adapter.md)、[設定ページ](../../docs/cookbook/adding-a-settings-card.md)のステップ・バイ・ステップ・ガイドを索引化しています。[Conversation サブシステム](../../docs/subsystems/conversation.md)は Chat ノードの組み立てを所有します。
