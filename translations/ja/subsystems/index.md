# サブシステム

日本語 | [English](../../../docs/subsystems/README.md) | [中文](../../../docs/subsystems/README.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/README.md](../../../docs/subsystems/README.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

DeepSeek Harness のサブシステムごとに 1 ページ：それが何であるか、それが動かすデータ構造、そして `ctx` サービスやイベント・スコープが背後にある場合は、そのサービスとイベントのリファレンスを運ぶ生成された **Cordis API** セクション。このフォルダは、サブシステムをまたぐ*振る舞い*（サービス・マップ、session／turn／step のライフサイクル、イベント分類）を説明する [architecture.md](../architecture.md) を補完する。ここでの各ページは、1 つのサブシステムの語彙と配線のリファレンスである。

| ページ | 所有するもの |
|---|---|
| [boot.md](boot.md) | 現在のプロファイルのプラグイン管理とランチャーのリロード調整 |
| [core.md](core.md) | `packages/core` が agent ループを制御する方法：パッケージごとのループ記述、agent の作成と所有（`AgentHandle`）、`Agent` ハンドルの配信／キャンセル／インターセプト契約、リポジトリ全体の型パターン（`…Map → derived-union`、ブランド付き id） |
| [llm-streaming.md](llm-streaming.md) | `packages/llm` の会話型：`Message`／`ContentBlock`、組み立てられたモデル要求、`StreamChunk` ワイヤー・プロトコルとアダプター契約、`BlockAssembler`、`LlmAdapter` プロバイダー契約 |
| [token-meter.md](token-meter.md) | 消費ログのリビジョンを伴う、不変のスカラーおよび位置ベースのリプレイ計測 |
| [scope.md](scope.md) | スコープ付き登録のアイデンティティ、ディスパッチ・キャリア、所有される `Scope` コンテキスト |
| [typert.md](typert.md) | Remote 呼び出し記述子、ルックアップ／Context 宣言、Typert レジストリ、Host Gateway／Client API の境界 |
| [goal.md](goal.md) | 永続化されたゴールのアイデンティティ、ライフサイクル・スナップショット、アクティブ化、変更レコード、ラウンドの帰属 |
| [schedule.md](schedule.md) | Host が所有するリマインダー・レコード、永続的な遷移、アクティブおよび終了ビュー、通常の会話への配信 |
| [todo.md](todo.md) | todo パッケージのリスト全体の項目型、永続イベントの所有権、投影 |
| [deliverables.md](deliverables.md) | ターンがユーザーに渡すもの：`present` からの `PresentedFile` 配信と、git スナップショットから得られる変更ファイルの Host 提供 `WorkspaceChangesSummary` |
| [commands.md](commands.md) | 人間コマンドのレジストリ・サービス：定義、アダプターの発見、直接呼び出し、結果、パース・ビュー |
| [session.md](session.md) | `SessionEventMap` の完全なバリアント・カタログ、`TurnEndReason`、`deriveMessages()`、実行の囲い込み、スタンドアロン・イベント |
| [working-directory.md](working-directory.md) | Session の実行ディレクトリ、永続的な変更、回復、ユーザー・コンテキスト |
| [worktrees.md](worktrees.md) | 既存の権限下での実験的な Git ブランチとチェックアウトの作成 |
| [forge.md](forge.md) | 実験的なコード・ホストの変更要求：プロバイダー契約、共有ステータス・ポーリング、Session バインディング |
| [persistence.md](persistence.md) | 永続化 seam：`SessionPersistence`、JSONL プロバイダー、`session/flush`、クラッシュ回復、`SessionHeader` |
| [settings.md](settings.md) | ユーザー設定 seam：`SettingsNamespace` の登録、階層的な解決（デフォルト → コンポジションの `base` → ユーザー文書）、owner スコープ、ホット・コミット |
| [credentials.md](credentials.md) | 資格情報 seam：設定内の `CredentialRef` 参照（値ではない）、操作ごとの解決、UI 安全な `CredentialInfo`、プロバイダーのソース・レイヤー |
| [session-query.md](session-query.md) | 論理レコード、境界付きの正確なイベント読み取り、関係トレース、セマンティック・フィルター／ドキュメント、全文検索結果ページ |
| [feedback.md](feedback.md) | ライフサイクルに束縛されたメッセージごとのフィードバック・レコード、楽観的バージョン、サイドカー永続化、Host Remote 契約 |
| [session-title.md](session-title.md) | 永続的なタイトル・スナップショット、引用されたソース・メッセージ seq、非同期プロバイダー契約 |
| [session-reference.md](session-reference.md) | 構造化されたセッション横断参照：`SessionReferenceInput`／`Candidate`、準備されたメッセージ・コンテキスト、安定したエラー分類 |
| [system-prompt.md](system-prompt.md) | アセンブリごとのコンテキスト、ツール・プロバイダーの結果、プロンプト・セクション、協調的アセンブリ |
| [tools.md](tools.md) | `ToolDefinition` の全フィールド、スキーマ DSL、`ToolExecution`／`ToolResult`、ツール・プレゼンテーションの UI 型、ガード付き実行パイプライン |
| [mcp.md](mcp.md) | 外部 MCP 接続、スコープ付きツールとリソース、サーバー指示、プロトコル結果、設定の所有権 |
| [user-questions.md](user-questions.md) | UI に支えられた人間の質問／回答 seam：`AskUserQuestionRequest`、回答／選択肢の語彙、プロバイダー API、エラー分類 |
| [approval.md](approval.md) | 1 回限りのユーザー承認 seam：`ApprovalRequest`、`ApprovalOutcome`、セッションごとのポリシー、監査イベント、回答者の契約 |
| [office-to-pdf.md](office-to-pdf.md) | 認可された Office から PDF への変換、ネイティブ／WASM エンジン、境界付きの共有再利用 |
| [attachment.md](attachment.md) | 永続的な画像のアイデンティティとメタデータ、検証の入力、検証済みの読み取り、`AttachmentStore` seam |
| [shell.md](shell.md) | シェル・エグゼキューター seam：`ShellExecRequest`／`Spec`、`ShellRunResult`、バックグラウンドの `ShellProcess` ハンドル |
| [subprocess.md](subprocess.md) | サブプロセス seam：完全に明示的な `SubprocessSpawnSpec`、オフセット・ベースの出力リーダー、未分類の `SubprocessOutcome`、管理された `DSH_*` 環境語彙 |
| [ssh.md](ssh.md) | POSIX SSH 接続と、リモートのファイルシステム、サブプロセス、サンドボックスのプロバイダー |
| [terminal.md](terminal.md) | 永続的なターミナル id、バックエンド／セッション契約、送信の準備性、境界付き読み取り、所有者可視のスナップショット |
| [sandbox.md](sandbox.md) | セッションごとのポリシー解決とプロセス閉じ込め seam：ファイル効果モード、実行／プロバイダー・ポリシー、`ConfinedArgv`、強制とフェイルクローズのエラー |
| [ptc-runtime.md](ptc-runtime.md) | PTC 実行 seam：`PtcRunRequest`／`Result`、バインディング名前空間、キャプチャされたログ、`PtcRunFailure` 分類 |
| [computer-use.md](computer-use.md) | 排他的な名前付き computer-use プロバイダー登録と Cua Driver 統合の選択肢 |
| [browser-use.md](browser-use.md) | 排他的な名前付き browser-use 登録、プロバイダーの選択肢、Session ごとのブラウザー所有権 |
| [extensions.md](extensions.md) | バージョン付きの動的 Cordis Plugins と Packages、Host／Client のアクティブ化、承認、ランタイム検査、ライフサイクルの解体 |
| [filesystem.md](filesystem.md) | ファイルシステム seam：`FsTarget`、読み取り／書き込み／編集の結果、観測ファイル状態、`FsErrorCode` |
| [lsp.md](lsp.md) | LSP ナビゲーション seam：`LspQueryRequest`／`Result`、`LspProvider`／`Service`、4 つの操作、`LspError` |
| [skills.md](skills.md) | スキル・サービス：発見の優先度、`SkillSummary`／`SkillDefinition`、セッション接頭辞カタログ、モデル向けの `skill` ロード |
| [compaction.md](compaction.md) | 圧縮 seam：`compaction/*` セッション・イベント、`CompactionResult`、`CompactionEngine` インターフェース |
| [subagent.md](subagent.md) | サブ agent seam：名前付きプロバイダー・レジストリ、`SubagentStartRequest`／`Result`／`Run`、開始時と実行時のケイパビリティ分割 |
| [voice-input.md](voice-input.md) | 実験的な名前付き認識器、一時的な音声、リビジョンでガードされたドラフト挿入 |
| [translation.md](translation.md) | 実験的な匿名テキスト翻訳と明示的なプロバイダー要求 |
| [agent-team.md](agent-team.md) | Agent Teams：暗黙の Lead アイデンティティ、名前付きで継続可能なチームメイト、直接のピア受信箱配信、共有タスク DAG |
| [liveshare.md](liveshare.md) | オプトインの LiveShare バンドル：ネストされた共有モード、フォークの上限と永続性、強制のインストール、共有 Session の外部駆動ターンが従う制約 |
| [claude-code-mods.md](claude-code-mods.md) | 実験的ブリッジ経由の Claude Code mods：すべてのイベント、`$` メンバー、チェーン規則、描画機能、Claude Code と異なるサンプル mod |
| [web.md](web.md) | Web アクセス seam：`WebSearchRequest`／`Result`、`WebFetchRequest`／`Result`、`WebFetchBody`、プロバイダーの可用性、`WebError` |
| [spill.md](spill.md) | スピル・ストレージ seam：`SaveTextSpill`、`SpillOwner`／`SpillSource`、`SpillRef`、ブランド付き `SpillLocator` |
| [workflow.md](workflow.md) | ワークフロー seam：`WorkflowStartRequest`、`WorkflowMeta`、`WorkflowRun`／`Result`、`workflow/*` イベント・ペイロード、`WorkflowError` の致命性 |
| [jobs.md](jobs.md) | バックグラウンド・ジョブのランタイム：ブランド付き `JobId`、プロデューサー契約、コンシューマー・ビュー、`ctx.jobs` サービスの振る舞い |
| [permission-presets.md](permission-presets.md) | permission プリセットのレイヤー：`PresetSpec`／`PresetOption`、導出された `custom` 状態、ログ専用の `permission/preset` イベント |
| [plan.md](plan.md) | プラン・モード：ログ専用の `plan/mode` 状態、保留中の選択のフラッシュ、`PlanModeConfig`、`exit_plan_mode` のレビュー弧 |
| [web-server.md](web-server.md) | HTTP キャリア：`WebRouteKind`／`WebRoute`、マッチ順序、クレーム可能なフォールバック席、インデックス・タップ |
| [webhook.md](webhook.md) | 認証済みのプロバイダー配信、任意のプログラム的ルール、ファイア・アンド・フォーゲットの Workspace Session 作成 |
| [storage.md](storage.md) | ストレージ・サブシステム：バックエンド契約（`StorageBackend`）、`StorageForms`、`DomainSpec`／`Domain`、`domain/changed` |
| [workspace.md](workspace.md) | ワークスペース・レジストリ：`Workspace`／`WorkspaceId`、登録と解決、session の `cwd` 関係 |
| [web-client.md](web-client.md) | ブラウザー・アーキテクチャ：ブート、Remote 通信、対になる Client モデル、UI アダプター、Conversation のアセンブリ、Slots、再接続のセマンティクス |
| [client-modules.md](client-modules.md) | Web プラグイン・テーブル：`dsh.client` 宣言、`WebBootGraph` のワイヤー・コンポジション、バンドル・ルートとインデックス・タップ |
| [slots.md](slots.md) | 型付き Web UI コンポジション：宣言の所有権、カーディナリティとスコープ、フレームワークと機能のインジェクション、props の導出、同梱の階層 |
| [client-resources.md](client-resources.md) | クライアント・リソース・モデル：`dsh-resource://<type>/…` アドレス、プロトコル・プロバイダーと `ResourceProtocolMap`、`useResource` グローバル・フックとその状態、ピンと解放 |
| [sidebar-right.md](sidebar-right.md) | 右 Sidebar：リソースとナビゲーションのアドレス、タブ型の登録とルーティング、`ctx.sidebarRight` ナビゲーション・サービス、ペイン・タブのスロットと owner props、リソース・モデル、Workspace Files サービス |
| [conversation.md](conversation.md) | ターゲット中立の Session イベント・アセンブリ：Context のアイデンティティ、Location データ、リプレイ・パス、ビュー・ビルダー、ターゲット所有のレンダー・ノード |
| [session-projection.md](session-projection.md) | 投影 seam：`SessionProjectionMap`、純粋な `ProjectionDefinition` ユニット、`ProjectionSnapshot` の一貫したカット、変更フィード |
| [session-telemetry.md](session-telemetry.md) | アウトバウンドの session レポート・ケイパビリティ seam：`SessionTelemetryRecord`／`SessionTelemetrySeverity`、`SessionTelemetrySink` 契約、`session-telemetry/record` のリダクト・ウォーターフォール |
| [otel.md](otel.md) | 共有の通常イベント・チャネルと Session ログ OTLP チャネル |
| [product-telemetry.md](product-telemetry.md) | 明示的な製品分析の送信と OTLP/HTTP トランスポート |

> これらのページの型宣言とその JSDoc はソースと等価であり、`pnpm run verify-type-equiv` によってドリフトが検査される（[development.md](../development.md#documenting-types-verbatim-ts-type-equiv) を参照）。通常のブロックは完全な宣言を保持し、`public-api` ブロックはボディを除去した public クラス宣言を保持する。Cordis のサービスとイベントは、各ページの生成された **Cordis API** セクションを使う。
