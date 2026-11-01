# DeepSeek Harness 日本語ドキュメント

> これは非公式の日本語訳です。英語版が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

DeepSeek Harness の英語ドキュメントを日本語に翻訳したローカル用の索引です。コード、コマンド、イベント名、設定キー、識別子は原文のまま残しています。

## 翻訳対象

- 翻訳済みドキュメント 276 ファイル（人手で書かれたドキュメントと、主要な生成ドキュメント）。
- `capability-seams`、`cli-help`、`event-producer-consumer`、`agent-lifecycle`、`tool-execution-pipeline` も含みます。

## 翻訳対象外

- 生成カタログ: [`config-catalog`](../../docs/config-catalog.md)、[`tool-catalog`](../../docs/tool-catalog.md)、[`persistence-catalog`](../../docs/persistence-catalog.md)、[`module-graph`](../../docs/module-graph.md) など。
- 凍結された過去フォーマット: [`persistence-changes/historical-formats`](../../docs/persistence-changes/historical-formats/)。
- i18n 契約: [`docs/i18n`](../../docs/i18n/)。

これらは生成物または内部管理文書であり、手翻訳すると陳腐化するため英語版を参照してください。

## 概要

| ドキュメント | 原文 |
|---|---|
| [DeepSeek Harness](overview.md) | `README.md` |

## 全体像・開発

| ドキュメント | 原文 |
|---|---|
| [AGENTS.md — ドキュメント標準](agents-doc.md) | `docs/AGENTS.md` |
| [エージェントのターンとステップのライフサイクル](agent-lifecycle.md) | `docs/agent-lifecycle.md` |
| [API ゲートウェイ](api-gateway.md) | `docs/api-gateway.md` |
| [DeepSeek Harness アーキテクチャ](architecture.md) | `docs/architecture.md` |
| [ケイパビリティ seam とコア・サービス](capability-seams.md) | `docs/capability-seams.md` |
| [dsh CLI ヘルプ](cli-help.md) | `docs/cli-help.md` |
| [Cordis 入門](cordis-primer.md) | `docs/cordis-primer.md` |
| [公式 DeepSeek LLM API のワイヤー拡張](deepseek-llm-api-wire-extensions.md) | `docs/deepseek-llm-api-wire-extensions.md` |
| [開発ガイド](development.md) | `docs/development.md` |
| [イベントの producer と consumer のマトリクス](event-producer-consumer.md) | `docs/event-producer-consumer.md` |
| [用語集](glossary.md) | `docs/glossary.md` |
| [Vendored パッケージの rescope](rescope.md) | `docs/rescope.md` |
| [Session 形式のバージョンとリリース・ステータス](session-format-status.md) | `docs/session-format-status.md` |
| [テスト・ポリシー](testing.md) | `docs/testing.md` |
| [ツール実行パイプライン](tool-execution-pipeline.md) | `docs/tool-execution-pipeline.md` |
| [DSH 統一角丸標準](ui-radius.md) | `docs/ui-radius.md` |
| [Web UI スタイル・リファレンス](web-styling.md) | `docs/web-styling.md` |

## ユーザーガイド

| ドキュメント | 原文 |
|---|---|
| [plugin の設定](user/develop/basic/config.md) | `docs/user/develop/basic/config.md` |
| [はじめての plugin](user/develop/basic/index.md) | `docs/user/develop/basic/index.md` |
| [plugin をパッケージ化してインストールする](user/develop/basic/publish.md) | `docs/user/develop/basic/publish.md` |
| [tool を構築する](user/develop/basic/tool.md) | `docs/user/develop/basic/tool.md` |
| [イベントシステム](user/develop/framework/events.md) | `docs/user/develop/framework/events.md` |
| [plugin とライフサイクル](user/develop/framework/index.md) | `docs/user/develop/framework/index.md` |
| [サービスと依存関係](user/develop/framework/service.md) | `docs/user/develop/framework/service.md` |
| [prompt から永続 plugin を設定する](user/develop/practice/dynamic-cordis.md) | `docs/user/develop/practice/dynamic-cordis.md` |
| [3ロールの capability 設計](user/develop/practice/index.md) | `docs/user/develop/practice/index.md` |
| [LLM アダプター](user/develop/practice/llm-adapter.md) | `docs/user/develop/practice/llm-adapter.md` |
| [GitHub webhook からレビュー用 Session を作成する](user/guide/github-review.md) | `docs/user/guide/github-review.md` |
| [Web UI を使う](user/guide/index.md) | `docs/user/guide/index.md` |
| [サードパーティの memory MCP サーバーを接続する](user/guide/mcp-memory.md) | `docs/user/guide/mcp-memory.md` |
| [ネットワークプロキシ経由で DSH を実行する](user/guide/network-proxy.md) | `docs/user/guide/network-proxy.md` |
| [plugin インデックス](user/guide/plugin-market.md) | `docs/user/guide/plugin-market.md` |
| [モデルを設定する](user/guide/providers.md) | `docs/user/guide/providers.md` |
| [リバースプロキシの背後で Web UI を公開する](user/guide/public-deployments.md) | `docs/user/guide/public-deployments.md` |
| [Python SDK を使い始める](user/guide/python-sdk.md) | `docs/user/guide/python-sdk.md` |
| [リマインダーをスケジュールする](user/guide/schedule.md) | `docs/user/guide/schedule.md` |
| [DeepSeek Harness](user/index.md) | `docs/user/index.md` |

## クックブック

| ドキュメント | 原文 |
|---|---|
| [クックブック: workspace パッケージの追加](cookbook/adding-a-package.md) | `docs/cookbook/adding-a-package.md` |
| [クックブック: プラグイン機能プレビューの追加](cookbook/adding-a-plugin-preview.md) | `docs/cookbook/adding-a-plugin-preview.md` |
| [クックブック: Remote API の追加](cookbook/adding-a-remote-api.md) | `docs/cookbook/adding-a-remote-api.md` |
| [クックブック: Session ログ形式バージョンの追加](cookbook/adding-a-session-format-version.md) | `docs/cookbook/adding-a-session-format-version.md` |
| [クックブック: ライブ設定フォーム](cookbook/adding-a-settings-card.md) | `docs/cookbook/adding-a-settings-card.md` |
| [ツール作成リファレンス](cookbook/adding-a-tool.md) | `docs/cookbook/adding-a-tool.md` |
| [クックブック: vendored パッケージの追加](cookbook/adding-a-vendored-package.md) | `docs/cookbook/adding-a-vendored-package.md` |
| [クックブック: LLM アダプターの追加](cookbook/adding-an-llm-adapter.md) | `docs/cookbook/adding-an-llm-adapter.md` |
| [クックブック: 拡張プラグインの形](cookbook/extension-cookbook.md) | `docs/cookbook/extension-cookbook.md` |
| [dsh-code-review スキルのメンテナンス](cookbook/maintaining-dsh-code-review.md) | `docs/cookbook/maintaining-dsh-code-review.md` |
| [スタック PR チェーンをまたぐレビューへの対応](cookbook/responding-to-pr-review-on-a-stack.md) | `docs/cookbook/responding-to-pr-review-on-a-stack.md` |
| [クックブック: 永続化型の変更をレビューする](cookbook/reviewing-persistence-type-changes.md) | `docs/cookbook/reviewing-persistence-type-changes.md` |

## Cordis チュートリアル

| ドキュメント | 原文 |
|---|---|
| [1. 最初のプラグイン](cordis-tutorial/01-first-plugin.md) | `docs/cordis-tutorial/01-first-plugin.md` |
| [2. ライフサイクルと効果](cordis-tutorial/02-lifecycle-and-effects.md) | `docs/cordis-tutorial/02-lifecycle-and-effects.md` |
| [3. サービス](cordis-tutorial/03-services.md) | `docs/cordis-tutorial/03-services.md` |
| [4. イベント](cordis-tutorial/04-events.md) | `docs/cordis-tutorial/04-events.md` |
| [5. 設定](cordis-tutorial/05-config.md) | `docs/cordis-tutorial/05-config.md` |
| [6. 構成と HMR](cordis-tutorial/06-composition-and-hmr.md) | `docs/cordis-tutorial/06-composition-and-hmr.md` |
| [7. harness へ](cordis-tutorial/07-into-the-harness.md) | `docs/cordis-tutorial/07-into-the-harness.md` |
| [Cordis チュートリアル](cordis-tutorial/index.md) | `docs/cordis-tutorial/index.md` |

## サブシステム

| ドキュメント | 原文 |
|---|---|
| [サブシステム](subsystems/index.md) | `docs/subsystems/README.md` |
| [Agent チーム](subsystems/agent-team.md) | `docs/subsystems/agent-team.md` |
| [ユーザー承認](subsystems/approval.md) | `docs/subsystems/approval.md` |
| [永続的な添付ファイル](subsystems/attachment.md) | `docs/subsystems/attachment.md` |
| [プロファイル管理](subsystems/boot.md) | `docs/subsystems/boot.md` |
| [ブラウザー操作](subsystems/browser-use.md) | `docs/subsystems/browser-use.md` |
| [Claude Code mods: 互換性](subsystems/claude-code-mods.md) | `docs/subsystems/claude-code-mods.md` |
| [クライアント・モジュール](subsystems/client-modules.md) | `docs/subsystems/client-modules.md` |
| [クライアント・リソース](subsystems/client-resources.md) | `docs/subsystems/client-resources.md` |
| [人間のコマンド](subsystems/commands.md) | `docs/subsystems/commands.md` |
| [コンパクション](subsystems/compaction.md) | `docs/subsystems/compaction.md` |
| [コンピューター操作](subsystems/computer-use.md) | `docs/subsystems/computer-use.md` |
| [Conversation の組み立て](subsystems/conversation.md) | `docs/subsystems/conversation.md` |
| [コア](subsystems/core.md) | `docs/subsystems/core.md` |
| [ユーザー認証情報](subsystems/credentials.md) | `docs/subsystems/credentials.md` |
| [成果物](subsystems/deliverables.md) | `docs/subsystems/deliverables.md` |
| [拡張機能](subsystems/extensions.md) | `docs/subsystems/extensions.md` |
| [メッセージフィードバック](subsystems/feedback.md) | `docs/subsystems/feedback.md` |
| [ファイルシステム](subsystems/filesystem.md) | `docs/subsystems/filesystem.md` |
| [Forge](subsystems/forge.md) | `docs/subsystems/forge.md` |
| [同一 Session のゴール](subsystems/goal.md) | `docs/subsystems/goal.md` |
| [バックグラウンド・ジョブ・ランタイム](subsystems/jobs.md) | `docs/subsystems/jobs.md` |
| [LiveShare](subsystems/liveshare.md) | `docs/subsystems/liveshare.md` |
| [LLM ストリーミング](subsystems/llm-streaming.md) | `docs/subsystems/llm-streaming.md` |
| [LSP ナビゲーション](subsystems/lsp.md) | `docs/subsystems/lsp.md` |
| [MCP](subsystems/mcp.md) | `docs/subsystems/mcp.md` |
| [Office から PDF へ](subsystems/office-to-pdf.md) | `docs/subsystems/office-to-pdf.md` |
| [OTel レポート](subsystems/otel.md) | `docs/subsystems/otel.md` |
| [権限プリセット](subsystems/permission-presets.md) | `docs/subsystems/permission-presets.md` |
| [セッション永続化](subsystems/persistence.md) | `docs/subsystems/persistence.md` |
| [プラン・モード](subsystems/plan.md) | `docs/subsystems/plan.md` |
| [製品テレメトリー](subsystems/product-telemetry.md) | `docs/subsystems/product-telemetry.md` |
| [PTC ランタイム](subsystems/ptc-runtime.md) | `docs/subsystems/ptc-runtime.md` |
| [プロセス sandbox](subsystems/sandbox.md) | `docs/subsystems/sandbox.md` |
| [ホスト全体のスケジュール](subsystems/schedule.md) | `docs/subsystems/schedule.md` |
| [スコープ付き登録](subsystems/scope.md) | `docs/subsystems/scope.md` |
| [セッション投影](subsystems/session-projection.md) | `docs/subsystems/session-projection.md` |
| [セッション・クエリ](subsystems/session-query.md) | `docs/subsystems/session-query.md` |
| [Session 参照](subsystems/session-reference.md) | `docs/subsystems/session-reference.md` |
| [セッションテレメトリ（SessionTelemetryBackend）](subsystems/session-telemetry.md) | `docs/subsystems/session-telemetry.md` |
| [セッションタイトル](subsystems/session-title.md) | `docs/subsystems/session-title.md` |
| [セッション](subsystems/session.md) | `docs/subsystems/session.md` |
| [プラグイン設定フォーム](subsystems/settings.md) | `docs/subsystems/settings.md` |
| [シェル・エグゼキューター](subsystems/shell.md) | `docs/subsystems/shell.md` |
| [右サイドバー](subsystems/sidebar-right.md) | `docs/subsystems/sidebar-right.md` |
| [スキル](subsystems/skills.md) | `docs/subsystems/skills.md` |
| [Web Client スロット](subsystems/slots.md) | `docs/subsystems/slots.md` |
| [スピル・ストレージ](subsystems/spill.md) | `docs/subsystems/spill.md` |
| [SSH](subsystems/ssh.md) | `docs/subsystems/ssh.md` |
| [ストレージ](subsystems/storage.md) | `docs/subsystems/storage.md` |
| [サブエージェント](subsystems/subagent.md) | `docs/subsystems/subagent.md` |
| [サブプロセス](subsystems/subprocess.md) | `docs/subsystems/subprocess.md` |
| [システムプロンプトの組み立て](subsystems/system-prompt.md) | `docs/subsystems/system-prompt.md` |
| [永続 PTY セッション](subsystems/terminal.md) | `docs/subsystems/terminal.md` |
| [Todo](subsystems/todo.md) | `docs/subsystems/todo.md` |
| [トークン・メーター](subsystems/token-meter.md) | `docs/subsystems/token-meter.md` |
| [ツール](subsystems/tools.md) | `docs/subsystems/tools.md` |
| [テキスト翻訳](subsystems/translation.md) | `docs/subsystems/translation.md` |
| [Typert リモート呼び出し](subsystems/typert.md) | `docs/subsystems/typert.md` |
| [ユーザー・インタラクション](subsystems/user-questions.md) | `docs/subsystems/user-questions.md` |
| [音声入力](subsystems/voice-input.md) | `docs/subsystems/voice-input.md` |
| [Web Client アーキテクチャ](subsystems/web-client.md) | `docs/subsystems/web-client.md` |
| [HTTP サーバー](subsystems/web-server.md) | `docs/subsystems/web-server.md` |
| [Web アクセス](subsystems/web.md) | `docs/subsystems/web.md` |
| [Webhook ランタイム](subsystems/webhook.md) | `docs/subsystems/webhook.md` |
| [ワークフロー](subsystems/workflow.md) | `docs/subsystems/workflow.md` |
| [作業ディレクトリ](subsystems/working-directory.md) | `docs/subsystems/working-directory.md` |
| [ワークスペース](subsystems/workspace.md) | `docs/subsystems/workspace.md` |
| [Git worktree](subsystems/worktrees.md) | `docs/subsystems/worktrees.md` |

## 防御的パターン

| ドキュメント | 原文 |
|---|---|
| [防御的パターン](defensive-patterns/index.md) | `docs/defensive-patterns/README.md` |
| [Client コンポーネントのルール](defensive-patterns/client-components.md) | `docs/defensive-patterns/client-components.md` |
| [会話と Session のルール](defensive-patterns/conversation-and-session.md) | `docs/defensive-patterns/conversation-and-session.md` |
| [防御的パターン](defensive-patterns/general.md) | `docs/defensive-patterns/general.md` |
| [パッケージ・サーフェスのルール](defensive-patterns/package-surface.md) | `docs/defensive-patterns/package-surface.md` |
| [Remote と API のルール](defensive-patterns/remote-and-api.md) | `docs/defensive-patterns/remote-and-api.md` |

## 永続化の変更履歴

| ドキュメント | 原文 |
|---|---|
| [初期の永続化型ベースライン](persistence-changes/2026-09-11-initial.md) | `docs/persistence-changes/2026-09-11-initial.md` |
| [2026-09-12-auto-review-error-metadata](persistence-changes/2026-09-12-auto-review-error-metadata.md) | `docs/persistence-changes/2026-09-12-auto-review-error-metadata.md` |
| [2026-09-13-working-directory](persistence-changes/2026-09-13-working-directory.md) | `docs/persistence-changes/2026-09-13-working-directory.md` |
| [2026-09-14-image-offload](persistence-changes/2026-09-14-image-offload.md) | `docs/persistence-changes/2026-09-14-image-offload.md` |
| [2026-09-14-workspace-changes-event](persistence-changes/2026-09-14-workspace-changes-event.md) | `docs/persistence-changes/2026-09-14-workspace-changes-event.md` |
| [2026-09-16-session-format-v4](persistence-changes/2026-09-16-session-format-v4.md) | `docs/persistence-changes/2026-09-16-session-format-v4.md` |
| [2026-09-18-schedule-optional-title](persistence-changes/2026-09-18-schedule-optional-title.md) | `docs/persistence-changes/2026-09-18-schedule-optional-title.md` |
| [2026-09-19-external-subagent-catalog](persistence-changes/2026-09-19-external-subagent-catalog.md) | `docs/persistence-changes/2026-09-19-external-subagent-catalog.md` |
| [2026-09-20-unknown-child-catalog](persistence-changes/2026-09-20-unknown-child-catalog.md) | `docs/persistence-changes/2026-09-20-unknown-child-catalog.md` |
| [2026-09-21-user-question-reply](persistence-changes/2026-09-21-user-question-reply.md) | `docs/persistence-changes/2026-09-21-user-question-reply.md` |
| [2026-09-30-job-session-lifecycle](persistence-changes/2026-09-30-job-session-lifecycle.md) | `docs/persistence-changes/2026-09-30-job-session-lifecycle.md` |
| [2026-10-05-ptc-operation-metadata](persistence-changes/2026-10-05-ptc-operation-metadata.md) | `docs/persistence-changes/2026-10-05-ptc-operation-metadata.md` |
| [2026-10-05-working-directory-attribution](persistence-changes/2026-10-05-working-directory-attribution.md) | `docs/persistence-changes/2026-10-05-working-directory-attribution.md` |
| [2026-10-07-title-reasoning-effort](persistence-changes/2026-10-07-title-reasoning-effort.md) | `docs/persistence-changes/2026-10-07-title-reasoning-effort.md` |
| [2026-10-13-deepseek-search-thinking](persistence-changes/2026-10-13-deepseek-search-thinking.md) | `docs/persistence-changes/2026-10-13-deepseek-search-thinking.md` |
| [2026-10-13-liveshare-share-viewer-principal](persistence-changes/2026-10-13-liveshare-share-viewer-principal.md) | `docs/persistence-changes/2026-10-13-liveshare-share-viewer-principal.md` |
| [2026-10-13-user-rpc-principal](persistence-changes/2026-10-13-user-rpc-principal.md) | `docs/persistence-changes/2026-10-13-user-rpc-principal.md` |
| [2026-10-22-action-card-status](persistence-changes/2026-10-22-action-card-status.md) | `docs/persistence-changes/2026-10-22-action-card-status.md` |
| [2026-10-25-skill-catalog-incomplete](persistence-changes/2026-10-25-skill-catalog-incomplete.md) | `docs/persistence-changes/2026-10-25-skill-catalog-incomplete.md` |
| [永続化型の変更記録](persistence-changes/index.md) | `docs/persistence-changes/README.md` |
| [DSH プレリリース間の永続化の変更](persistence-changes/releases/index.md) | `docs/persistence-changes/releases/README.md` |
| [永続化リリース: dsh-v0.0.1-rc.1](persistence-changes/releases/dsh-v0.0.1-rc.1.md) | `docs/persistence-changes/releases/dsh-v0.0.1-rc.1.md` |
| [永続化リリース: dsh-v0.0.1-rc.2](persistence-changes/releases/dsh-v0.0.1-rc.2.md) | `docs/persistence-changes/releases/dsh-v0.0.1-rc.2.md` |
| [永続化リリース: dsh-v0.0.1-rc.3](persistence-changes/releases/dsh-v0.0.1-rc.3.md) | `docs/persistence-changes/releases/dsh-v0.0.1-rc.3.md` |
| [永続化リリース: dsh-v0.0.1-rc.4](persistence-changes/releases/dsh-v0.0.1-rc.4.md) | `docs/persistence-changes/releases/dsh-v0.0.1-rc.4.md` |
| [永続化リリース: dsh-v0.0.1-rc.5](persistence-changes/releases/dsh-v0.0.1-rc.5.md) | `docs/persistence-changes/releases/dsh-v0.0.1-rc.5.md` |
| [永続化リリース: dsh-v0.1.0-rc.1](persistence-changes/releases/dsh-v0.1.0-rc.1.md) | `docs/persistence-changes/releases/dsh-v0.1.0-rc.1.md` |
| [永続化リリース: dsh-v0.1.0-rc.2](persistence-changes/releases/dsh-v0.1.0-rc.2.md) | `docs/persistence-changes/releases/dsh-v0.1.0-rc.2.md` |
| [永続化リリース: dsh-v0.1.0-rc.3](persistence-changes/releases/dsh-v0.1.0-rc.3.md) | `docs/persistence-changes/releases/dsh-v0.1.0-rc.3.md` |
| [永続化リリース: dsh-v0.1.0-rc.5](persistence-changes/releases/dsh-v0.1.0-rc.5.md) | `docs/persistence-changes/releases/dsh-v0.1.0-rc.5.md` |
| [永続化リリース: dsh-v0.1.0-rc.6](persistence-changes/releases/dsh-v0.1.0-rc.6.md) | `docs/persistence-changes/releases/dsh-v0.1.0-rc.6.md` |
| [永続化リリース: dsh-v0.1.0-rc.7](persistence-changes/releases/dsh-v0.1.0-rc.7.md) | `docs/persistence-changes/releases/dsh-v0.1.0-rc.7.md` |
| [永続化リリース: dsh-v0.1.0-rc.8](persistence-changes/releases/dsh-v0.1.0-rc.8.md) | `docs/persistence-changes/releases/dsh-v0.1.0-rc.8.md` |
| [永続化リリース: dsh-v0.1.1-rc.1](persistence-changes/releases/dsh-v0.1.1-rc.1.md) | `docs/persistence-changes/releases/dsh-v0.1.1-rc.1.md` |
| [永続化リリース: dsh-v0.1.1-rc.2](persistence-changes/releases/dsh-v0.1.1-rc.2.md) | `docs/persistence-changes/releases/dsh-v0.1.1-rc.2.md` |
| [永続化リリース: dsh-v0.1.2-alpha.1](persistence-changes/releases/dsh-v0.1.2-alpha.1.md) | `docs/persistence-changes/releases/dsh-v0.1.2-alpha.1.md` |
| [永続化リリース: dsh-v0.1.2-alpha.2](persistence-changes/releases/dsh-v0.1.2-alpha.2.md) | `docs/persistence-changes/releases/dsh-v0.1.2-alpha.2.md` |
| [永続化リリース: dsh-v0.1.2-alpha.3](persistence-changes/releases/dsh-v0.1.2-alpha.3.md) | `docs/persistence-changes/releases/dsh-v0.1.2-alpha.3.md` |
| [永続化リリース: dsh-v0.1.2-alpha.4](persistence-changes/releases/dsh-v0.1.2-alpha.4.md) | `docs/persistence-changes/releases/dsh-v0.1.2-alpha.4.md` |
| [永続化リリース: dsh-v0.1.2-alpha.5](persistence-changes/releases/dsh-v0.1.2-alpha.5.md) | `docs/persistence-changes/releases/dsh-v0.1.2-alpha.5.md` |
| [永続化リリース: dsh-v0.1.2-rc.1](persistence-changes/releases/dsh-v0.1.2-rc.1.md) | `docs/persistence-changes/releases/dsh-v0.1.2-rc.1.md` |
| [永続化リリース: dsh-v0.1.3-alpha.1](persistence-changes/releases/dsh-v0.1.3-alpha.1.md) | `docs/persistence-changes/releases/dsh-v0.1.3-alpha.1.md` |
| [永続化リリース: dsh-v0.1.3-alpha.2](persistence-changes/releases/dsh-v0.1.3-alpha.2.md) | `docs/persistence-changes/releases/dsh-v0.1.3-alpha.2.md` |
| [永続化リリース: dsh-v0.1.5-alpha.1](persistence-changes/releases/dsh-v0.1.5-alpha.1.md) | `docs/persistence-changes/releases/dsh-v0.1.5-alpha.1.md` |
| [永続化リリース: dsh-v0.1.5-alpha.2](persistence-changes/releases/dsh-v0.1.5-alpha.2.md) | `docs/persistence-changes/releases/dsh-v0.1.5-alpha.2.md` |
| [永続化リリース: dsh-v0.1.5-rc.1](persistence-changes/releases/dsh-v0.1.5-rc.1.md) | `docs/persistence-changes/releases/dsh-v0.1.5-rc.1.md` |
| [永続化リリース: dsh-v0.1.5-rc.2](persistence-changes/releases/dsh-v0.1.5-rc.2.md) | `docs/persistence-changes/releases/dsh-v0.1.5-rc.2.md` |

## アップグレードガイド

| ドキュメント | 原文 |
|---|---|
| [Schedule は Automation tasks オプション bundle に移る](upgrade-guide/v0.1.7-rc.2/schedule-optional-bundle/guide.md) | `docs/upgrade-guide/v0.1.7-rc.2/schedule-optional-bundle/guide.md` |
| [レガシーな normal の作業詳細は Detailed として表示される](upgrade-guide/v0.1.7-rc.2/transcript-view-legacy-normal/guide.md) | `docs/upgrade-guide/v0.1.7-rc.2/transcript-view-legacy-normal/guide.md` |
| [account サインインのエラーコード](upgrade-guide/v0.2.0-rc.2/account-sign-in-errors/guide.md) | `docs/upgrade-guide/v0.2.0-rc.2/account-sign-in-errors/guide.md` |
| [NumPy と pandas の上流テストには別の環境が必要](upgrade-guide/v0.2.0-rc.2/bundled-python-tests/guide.md) | `docs/upgrade-guide/v0.2.0-rc.2/bundled-python-tests/guide.md` |
| [カスタム Python runtime の alias は sibling interpreter を指さなければならない](upgrade-guide/v0.2.0-rc.2/primary-runtime-python-aliases/guide.md) | `docs/upgrade-guide/v0.2.0-rc.2/primary-runtime-python-aliases/guide.md` |
| [runtime invariant plugin は削除される](upgrade-guide/v0.2.0-rc.2/remove-runtime-invariants/guide.md) | `docs/upgrade-guide/v0.2.0-rc.2/remove-runtime-invariants/guide.md` |
| [Automation tasks bundle の削除](upgrade-guide/v0.2.0-rc.2/schedule-bundle-retired/guide.md) | `docs/upgrade-guide/v0.2.0-rc.2/schedule-bundle-retired/guide.md` |
| [subpath plugin は <subpath>/package.json の代わりに <subpath>/icon を読む](upgrade-guide/v0.2.0-rc.2/subpath-plugin-display-manifest/guide.md) | `docs/upgrade-guide/v0.2.0-rc.2/subpath-plugin-display-manifest/guide.md` |
| [DeepSeek API-key のモデル可視性](upgrade-guide/v0.2.1-alpha.1/deepseek-model-discovery/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/deepseek-model-discovery/guide.md` |
| [ローカルで test update-policy authentication dialog を有効にする](upgrade-guide/v0.2.1-alpha.1/desktop-test-auth-popup/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/desktop-test-auth-popup/guide.md` |
| [オプション機能の明示的なインストールと構成](upgrade-guide/v0.2.1-alpha.1/experimental-package-composition/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/experimental-package-composition/guide.md` |
| [instruction のロードは1つの harness home を解決する](upgrade-guide/v0.2.1-alpha.1/instruction-home-override/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/instruction-home-override/guide.md` |
| [バックグラウンド job id は1つの実行を識別する](upgrade-guide/v0.2.1-alpha.1/job-id-identity/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/job-id-identity/guide.md` |
| [JSONL persistence をその plugin を通じてマウントする](upgrade-guide/v0.2.1-alpha.1/jsonl-persistence-plugin/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/jsonl-persistence-plugin/guide.md` |
| [Messages mock server から名前付き event を読む](upgrade-guide/v0.2.1-alpha.1/mock-messages-events/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/mock-messages-events/guide.md` |
| [ネイティブ subagent bundle はグローバル tool を含む](upgrade-guide/v0.2.1-alpha.1/native-subagent-bundle-tools/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/native-subagent-bundle-tools/guide.md` |
| [New Session で新しい Agent を開始する](upgrade-guide/v0.2.1-alpha.1/new-session-generations/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/new-session-generations/guide.md` |
| [dsh plugin のオプション配置](upgrade-guide/v0.2.1-alpha.1/plugin-option-placement/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/plugin-option-placement/guide.md` |
| [オプションの Python runtime リソースを明示的にダウンロードする](upgrade-guide/v0.2.1-alpha.1/python-optional-resources/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/python-optional-resources/guide.md` |
| [Python PTC は共有 file sandbox を適用する](upgrade-guide/v0.2.1-alpha.1/python-ptc-sandbox/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/python-ptc-sandbox/guide.md` |
| [カスタム title route を準備する](upgrade-guide/v0.2.1-alpha.1/session-title-adapters/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/session-title-adapters/guide.md` |
| [session-title provider を移行する](upgrade-guide/v0.2.1-alpha.1/session-title-provider-strategy/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/session-title-provider-strategy/guide.md` |
| [SSH helper と PTC の起動設定](upgrade-guide/v0.2.1-alpha.1/ssh-helper-launch/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/ssh-helper-launch/guide.md` |
| [委譲を管理された subagent activation へ移行する](upgrade-guide/v0.2.1-alpha.1/subagent-activations/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/subagent-activations/guide.md` |
| [preset を通じた subagent 構成の設定](upgrade-guide/v0.2.1-alpha.1/subagent-preset-composition/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/subagent-preset-composition/guide.md` |
| [Agent Teams のメッセージは target inbox を使う](upgrade-guide/v0.2.1-alpha.1/team-direct-inbox/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/team-direct-inbox/guide.md` |
| [terminal コマンドのラベル](upgrade-guide/v0.2.1-alpha.1/terminal-command-labels/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/terminal-command-labels/guide.md` |
| [native または PTC の tool presentation を選択する](upgrade-guide/v0.2.1-alpha.1/tool-presentation-mode/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/tool-presentation-mode/guide.md` |
| [Web listener と信頼設定は webStartup へ移る](upgrade-guide/v0.2.1-alpha.1/web-listener-trust-config/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/web-listener-trust-config/guide.md` |
| [working-directory 設定と tool presentation の移行](upgrade-guide/v0.2.1-alpha.1/working-directory/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.1/working-directory/guide.md` |
| [timer wrapper の戻り値型と disposal](upgrade-guide/v0.2.1-alpha.2/cordis-timer-wrappers/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.2/cordis-timer-wrappers/guide.md` |
| [account identity の変更を明示的に購読する](upgrade-guide/v0.2.1-alpha.2/deepseek-account-identity-watch/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.2/deepseek-account-identity-watch/guide.md` |
| [DeepSeek account のモデルは account listing から取得する](upgrade-guide/v0.2.1-alpha.2/deepseek-account-model-catalog/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.2/deepseek-account-model-catalog/guide.md` |
| [Desktop のフォントサイズショートカットはカスタムのキー組み合わせを予約する](upgrade-guide/v0.2.1-alpha.2/desktop-font-size-shortcuts/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.2/desktop-font-size-shortcuts/guide.md` |
| [--from-default-profile は既存の profile を置き換える](upgrade-guide/v0.2.1-alpha.2/from-default-profile-replaces-target/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.2/from-default-profile-replaces-target/guide.md` |
| [非 page client の interaction 参加を選択する](upgrade-guide/v0.2.1-alpha.2/operator-connect/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.2/operator-connect/guide.md` |
| [Azure provider route の改名](upgrade-guide/v0.2.1-alpha.2/pi-ai-azure-provider/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.2/pi-ai-azure-provider/guide.md` |
| [credential なしの pi-ai モデル可視性](upgrade-guide/v0.2.1-alpha.2/pi-ai-model-discovery/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.2/pi-ai-model-discovery/guide.md` |
| [HMR 下でインストール済み plugin パッケージへのインストールは拒否される](upgrade-guide/v0.2.1-alpha.2/plugin-install-over-installed/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.2/plugin-install-over-installed/guide.md` |
| [viewProfilePackage は出力するフィールドを受け取る](upgrade-guide/v0.2.1-alpha.2/plugin-manager-view-fields/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.2/plugin-manager-view-fields/guide.md` |
| [SDK-minimal の画像読み取りと自動 compaction](upgrade-guide/v0.2.1-alpha.2/sdk-minimal-image-compaction/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.2/sdk-minimal-image-compaction/guide.md` |
| [保存済みキーはエクスポートされた変数より優先される](upgrade-guide/v0.2.1-alpha.2/stored-credentials-precedence/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.2/stored-credentials-precedence/guide.md` |
| [plugin の互換性は engines.dsh を最初に読む](upgrade-guide/v0.2.1-alpha.3/plugin-engines-dsh/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.3/plugin-engines-dsh/guide.md` |
| [account 通知の consumer を更新する](upgrade-guide/v0.2.1-alpha.4/account-notification-fields/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.4/account-notification-fields/guide.md` |
| [モデル切り替えは移行先の context 圧力を確認する](upgrade-guide/v0.2.1-alpha.4/model-switch-compaction/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.4/model-switch-compaction/guide.md` |
| [Session のモデル選択は次の step に適用される](upgrade-guide/v0.2.1-alpha.4/session-model-selection/guide.md) | `docs/upgrade-guide/v0.2.1-alpha.4/session-model-selection/guide.md` |
| [@deepseek-ai/dsh-agent-instructions の maxBytes を instruction chain ごとに設定する](upgrade-guide/v0.2.1-rc.1/agent-instructions-chain-budgets/guide.md) | `docs/upgrade-guide/v0.2.1-rc.1/agent-instructions-chain-budgets/guide.md` |
| [subagent のモデル選択と制御 tool を更新する](upgrade-guide/v0.2.1-rc.1/subagent-control-tools/guide.md) | `docs/upgrade-guide/v0.2.1-rc.1/subagent-control-tools/guide.md` |
| [Approval は Projection registry を必要とする](upgrade-guide/v0.2.1-rc.2/approval-projection-dependency/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/approval-projection-dependency/guide.md` |
| [シャットダウン後に Auto review Session を再度開く](upgrade-guide/v0.2.1-rc.2/auto-review-shutdown/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/auto-review-shutdown/guide.md` |
| [Auto review は user 由来の命令を認識する](upgrade-guide/v0.2.1-rc.2/auto-review-user-sources/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/auto-review-user-sources/guide.md` |
| [組み込み言語は重複する言語パックを置き換える](upgrade-guide/v0.2.1-rc.2/built-in-japanese/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/built-in-japanese/guide.md` |
| [条件付き tool を step admission を通じて宣言する](upgrade-guide/v0.2.1-rc.2/conditional-tools/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/conditional-tools/guide.md` |
| [カスタム Client 構成における context menu サービス](upgrade-guide/v0.2.1-rc.2/context-menu-services/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/context-menu-services/guide.md` |
| [Computer Use の同意ストレージ](upgrade-guide/v0.2.1-rc.2/cua-consent-storage/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/cua-consent-storage/guide.md` |
| [CUA は PTC session scope をデフォルトにする](upgrade-guide/v0.2.1-rc.2/cua-preset-scope/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/cua-preset-scope/guide.md` |
| [CUA の相対座標は 0–999 を使う](upgrade-guide/v0.2.1-rc.2/cua-relative-coordinates/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/cua-relative-coordinates/guide.md` |
| [画像ビューに file preparation callback を提供する](upgrade-guide/v0.2.1-rc.2/file-drag-callbacks/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/file-drag-callbacks/guide.md` |
| [headless startup を置き換える場合の policy 設定](upgrade-guide/v0.2.1-rc.2/headless-policy-composition/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/headless-policy-composition/guide.md` |
| [ui-deepseek-model-label-1017 パッケージの削除](upgrade-guide/v0.2.1-rc.2/model-badge-package-removal/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/model-badge-package-removal/guide.md` |
| [New Session は blank Session に戻る](upgrade-guide/v0.2.1-rc.2/new-session-blank-reuse/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/new-session-blank-reuse/guide.md` |
| [Node.js 24 が必要](upgrade-guide/v0.2.1-rc.2/node-24/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/node-24/guide.md` |
| [bundle リストを @deepseek-ai/dsh-official-bundles から import する](upgrade-guide/v0.2.1-rc.2/official-bundle-lists/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/official-bundle-lists/guide.md` |
| [profile 設定を通じて persistent terminal を管理する](upgrade-guide/v0.2.1-rc.2/persistent-terminal-controls/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/persistent-terminal-controls/guide.md` |
| [pi-ai のデフォルトモデル容量](upgrade-guide/v0.2.1-rc.2/pi-ai-default-capacities/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/pi-ai-default-capacities/guide.md` |
| [pi-ai 1.3.0 のモデルカタログ変更](upgrade-guide/v0.2.1-rc.2/pi-ai-model-catalog/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/pi-ai-model-catalog/guide.md` |
| [pi-ai 1.3.0 のプロトコルと replay の変更](upgrade-guide/v0.2.1-rc.2/pi-ai-protocol-replay/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/pi-ai-protocol-replay/guide.md` |
| [Script プログラムと明示的な state retention を扱う](upgrade-guide/v0.2.1-rc.2/ptc-script-format/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/ptc-script-format/guide.md` |
| [SDK の desktop tool と context](upgrade-guide/v0.2.1-rc.2/sdk-desktop-context/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/sdk-desktop-context/guide.md` |
| [SDK-minimal は永続的な checkpoint を必要とする](upgrade-guide/v0.2.1-rc.2/sdk-minimal-checkpoints/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/sdk-minimal-checkpoints/guide.md` |
| [Session projection 登録ハンドル](upgrade-guide/v0.2.1-rc.2/session-projection-handles/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/session-projection-handles/guide.md` |
| [store 所有の Session 書き込みは公開に従う](upgrade-guide/v0.2.1-rc.2/session-store-writes/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/session-store-writes/guide.md` |
| [tool executor をマウントする](upgrade-guide/v0.2.1-rc.2/tool-executor-plugins/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/tool-executor-plugins/guide.md` |
| [web_fetch はプライベートな宛先に到達する](upgrade-guide/v0.2.1-rc.2/web-fetch-private-destinations/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/web-fetch-private-destinations/guide.md` |
| [WebServer の index 描画は Peer を取る](upgrade-guide/v0.2.1-rc.2/webserver-index-peer/guide.md) | `docs/upgrade-guide/v0.2.1-rc.2/webserver-index-peer/guide.md` |
| [画像読み取りリクエストは完全な attachment reference を含む](upgrade-guide/v0.2.1-rc.3/attachment-read-access/guide.md) | `docs/upgrade-guide/v0.2.1-rc.3/attachment-read-access/guide.md` |
| [Computer Use helper を認可する](upgrade-guide/v0.2.1-rc.3/computer-use-helper-permissions/guide.md) | `docs/upgrade-guide/v0.2.1-rc.3/computer-use-helper-permissions/guide.md` |
| [macOS Computer Use のアプリケーション identity](upgrade-guide/v0.2.1-rc.3/cua-bundle-id-consent/guide.md) | `docs/upgrade-guide/v0.2.1-rc.3/cua-bundle-id-consent/guide.md` |
| [生成される worktree 名の consumer を更新する](upgrade-guide/v0.2.1-rc.3/generated-worktree-names/guide.md) | `docs/upgrade-guide/v0.2.1-rc.3/generated-worktree-names/guide.md` |
| [通常の fork は独立した Session になる](upgrade-guide/v0.2.1-rc.3/independent-session-forks/guide.md) | `docs/upgrade-guide/v0.2.1-rc.3/independent-session-forks/guide.md` |
| [PTC の caller cancellation は標準の tool エラーコードを使う](upgrade-guide/v0.2.1-rc.3/ptc-caller-cancellation/guide.md) | `docs/upgrade-guide/v0.2.1-rc.3/ptc-caller-cancellation/guide.md` |
| [Session を follow するときに activation を要求する](upgrade-guide/v0.2.1-rc.3/session-follow-activation/guide.md) | `docs/upgrade-guide/v0.2.1-rc.3/session-follow-activation/guide.md` |
| [Web でのファイルダウンロード](upgrade-guide/v0.2.1-rc.3/web-file-downloads/guide.md) | `docs/upgrade-guide/v0.2.1-rc.3/web-file-downloads/guide.md` |
| [モデル一覧は自動的に更新される](upgrade-guide/v0.2.1-rc.4/model-listing-refresh/guide.md) | `docs/upgrade-guide/v0.2.1-rc.4/model-listing-refresh/guide.md` |
| [Reasoning Body Slot のテキストフィールド](upgrade-guide/v0.2.1-rc.4/reasoning-body-props/guide.md) | `docs/upgrade-guide/v0.2.1-rc.4/reasoning-body-props/guide.md` |
| [context menu の型とアイコン](upgrade-guide/v0.2.2-alpha.2/context-menu-target/guide.md) | `docs/upgrade-guide/v0.2.2-alpha.2/context-menu-target/guide.md` |
| [Forge の Git クエリ権限と content conversion](upgrade-guide/v0.2.2-alpha.2/forge-git-permissions/guide.md) | `docs/upgrade-guide/v0.2.2-alpha.2/forge-git-permissions/guide.md` |
| [補助的な LLM リクエストの purpose を宣言する](upgrade-guide/v0.2.2-alpha.2/llm-request-purpose-declarations/guide.md) | `docs/upgrade-guide/v0.2.2-alpha.2/llm-request-purpose-declarations/guide.md` |
| [Schedule tool は RRULE と target ですべての Host task を管理する](upgrade-guide/v0.2.2-alpha.2/schedule-agent-tools/guide.md) | `docs/upgrade-guide/v0.2.2-alpha.2/schedule-agent-tools/guide.md` |
| [Schedule は Host 全体の単一管理 API を使う](upgrade-guide/v0.2.2-alpha.2/schedule-remote-host-tasks/guide.md) | `docs/upgrade-guide/v0.2.2-alpha.2/schedule-remote-host-tasks/guide.md` |
| [SDK run はネイティブコマンドを受け付ける](upgrade-guide/v0.2.2-alpha.2/sdk-run-commands/guide.md) | `docs/upgrade-guide/v0.2.2-alpha.2/sdk-run-commands/guide.md` |

## ポストモーテム

| ドキュメント | 原文 |
|---|---|
| [ポストモーテム 0001: 接続時に ACP サーバーがクラッシュ — export default がプラグインの inject を落とした](postmortem/0001-acp-default-export-drops-inject.md) | `docs/postmortem/0001-acp-default-export-drops-inject.md` |
| [ポストモーテム 0002: ファイルシステムのスナップショットツールが恒久的に無効化された](postmortem/0002-js-expression-disabled-filesystem-tools.md) | `docs/postmortem/0002-js-expression-disabled-filesystem-tools.md` |
| [ポストモーテム 0003: Web agent が現在の GUI ではなく代替サーバーを検証した](postmortem/0003-web-agent-gui-feedback-loop.md) | `docs/postmortem/0003-web-agent-gui-feedback-loop.md` |
| [ポストモーテム 0004: Landlock の部分適用通知が子プロセスの失敗を誤分類した](postmortem/0004-landlock-partial-notice-misclassified-child-failures.md) | `docs/postmortem/0004-landlock-partial-notice-misclassified-child-failures.md` |
| [ポストモーテム](postmortem/index.md) | `docs/postmortem/README.md` |

## 翻訳の方針

- 上流の i18n 契約は、`docs/**` と README に対して en/zh の3ファイルペア（`foo.md` / `foo.zh.md` / `foo.i18n.yaml`）を要求し、locale ディレクトリを禁止しています。
- この翻訳はスキャン対象外の `translations/ja/` に置いた非公式訳で、上流の PR にはそのまま出しません。
- 英語版を正とし、日本語版は参照用の非公式訳とします。
- コード、コマンド、イベント名、型名、設定キーは翻訳しません。
- 原文の更新に追従できるよう、翻訳ファイルは英語ファイルと分けて管理します。
