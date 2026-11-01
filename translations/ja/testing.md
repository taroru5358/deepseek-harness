# テスト・ポリシー

日本語 | [English](../../docs/testing.md) | [中文](../../docs/testing.zh.md)

> これは非公式の日本語訳です。英語版（[docs/testing.md](../../docs/testing.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

このリポジトリが階層ごとにどうテストするか、また緑のスイートを意味のあるものに保つルールを説明します。コマンドはルートの [AGENTS.md](../../AGENTS.md) にあり、根拠はリンク先の Agent Notes が担います。

## 階層

- **ユニット**（`pnpm run test`）: `tests/**` 配下のパッケージとサンプルの spec、および `scripts/**/*.spec.ts` 配下のリポジトリ・スクリプトの spec を vitest で実行します。テストは、それが検証するコード領域のそばに置きます。すべてのレジストリには HMR 安全性のテスト（寄与する fiber を dispose し、クリーンアップを検証する）を用意します。エッジケース、エラーパス、イベントの順序、並行処理の競合、契約の回帰に対する恒久的なテストを優先します（`packages/core/agent-loop/tests/contract-regressions.spec.ts` を参照）。
- **カバレッジ・ゲート**（`pnpm run test:coverage`）: ゲートとなる実行で、`packages/*/*/src` のファイルごとに 100% を要求します。カバーされていない行は、テストを後付けするのではなく、ゲートが削除対象として指摘するデッドコードであることがよくあります。行カバレッジは必要ですが、決して十分ではありません。行が実行されたことしか証明せず、機能が出荷どおりに動くことは証明しません。`packages/shell/pwsh-local/src` のファイルごとの 100% には実際の `pwsh` が必要です。ない場合、その executor スイートは自己スキップし、`vitest.config.ts` がそのファイルを除外するため、pwsh のないホストでも緑のままです。一方、CI ランナーは pwsh を同梱し、完全な基準を適用します。
- **実 API e2e**（`pnpm run test:e2e`）: 実際のプロバイダー API に対するキー付きテストで、DeepSeek モデルに加え、それぞれ自前のキー（`EXA_API_KEY`、`PERPLEXITY_API_KEY`、…）でゲートするプロバイダー固有のスモークを含みます。各スイートはキーがなければ自己スキップし、キーのない CI は緑のままです（[実 API e2e の Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/testing/2026-06-19-real-api-e2e-ci.md)）。
- **オーナー・ローカルの期待出力**（`pnpm run test:expected`）: 記録済みセッションの往復なしで、組み立て済みの CLI／プロセスの期待値をキーなしで検証します。ドライバーは `tests/expected/` の隣で `*.expected.e2e.ts` を使います。CI はビルド済みのエクスポートを実行します。パッケージ／スクリプトの期待値には `test`、ブラウザの期待値には `test:web` を使います。
- **パフォーマンス・ベンチマーク**（`pnpm run test:bench`、必須の Linux PR ゲート `node 24 / benchmarks`）: `benchmarks/` がユーザーパスのゲートをまとめます。ライブラリとワーカーをビルドし、計時対象のコードは素の Node で実行し、TSX では実行しません。合成入力が時間、ヒープ、スケーリングの予算を強制し、パッケージ・ローカルの `.perf.ts` は診断用に留まります（[ルール](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/testing/2026-09-04-session-open-performance-gate.md)）。
- **スナップショット**（`pnpm run test:snapshot`）: トップレベルのシナリオについて、記録済みの最上位の親世代がユーザー入力とモデルのリプレイを供給し、その後、期待される永続化結果として機能します。親のファイル名は `session[.vN].jsonl`、子の役割は `session.<ordinal>[.vN].jsonl` です。v0 は `.v0` を省略し、正のバージョンには小文字の `.vN` が必要で、各ファイル名はそのヘッダーと一致しなければなりません。プロセスのシナリオは `dsh` を通じて起動します。headless はワンショットの挙動、SDK は永続的な制御、ACP は自動化プロトコルの挙動、Web は同じ Session のそばにブラウザ／ARIA のエビデンスを保持します。`snapshot.yml` はプロファイル、コンポジション／ヘッダーのクラス、記録ポリシー、例外的なリプレイや入力のメタデータ、ワークスペースの事実を宣言します。型付きトークンが親子の同一性関係を保ち、ヘッダーの固定だけがプロンプト／スキーマのサイドカーを所有します。変更を伴うシナリオは、完全な `workspace.expected/` ツリーを独立に比較し、記録とリフレッシュはこのツリーを書き換えません。モデルのトランスクリプトが変わったときは `test:snapshot:record`、リプレイ入力がまだ有効なときは `test:snapshot:refresh` を使い、生じた差分はすべてレビューします。
- **Web ブラウザ・スナップショット**（`pnpm run test:web`、必須の Linux PR ゲート）: コマンドはプラグインの CSS をビルドした後、セッション駆動の `snapshots/web/` と UI のみの `apps/web/tests/expected/` の出力をすべて Chromium で比較します。モデル／推論のピッカーは WebKit でも実行します。CI は読み取り専用の `DSH_SNAPSHOT=replay` を強制し、記録／リフレッシュはローカルに留め、すべての差分をレビューします（[Web レーン](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/testing/2026-07-24-web-gui-browser-e2e-lane.md)、[CI リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/run-web-snapshots.ts)）。

セッション・フィクスチャはヘッダーとペイロードを保持しますが、ボディのシーケンス／時刻エンベロープは省略し、リプレイがそれを合成します。リプレイ、記録、リフレッシュは、親子それぞれの役割で最も高い世代を選びます。現在のフィクスチャはファイル名とヘッダーに[ライター形式](session-format-status.md)を使い、1 イベント 1 行で、コンパクトな Assistant ストリームを埋め込みます。履歴フィクスチャはリリース時の表現を保ち、明示的な `sessionFormat` の所有者が移行のカバレッジを保ちます。先行するものを変えずに後続を追加するには、[形式バージョンのクックブック](cookbook/adding-a-session-format-version.md#snapshot-successors)に従います。

## spec の実行方法

フォークされたワーカーは複数の spec ファイルを同時に実行し、カバレッジ・ゲートはそのジョブ内の他のゲートと並んで[並行パーティション](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/coverage-partitions.ts)に分割され、セルフホスト・ランナーは 1 つのホストと 1 つのボリュームを共有します。分離されるのはプロセスだけであり、ポート、予測可能なパス、外部ネームスペース、継承した子は分離されません。獲得した各リソースは teardown を通じて所有します。単独でのみ通る spec は、不安定なランナーではなくテストの欠陥です。[dsh-ci-test-reliability](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/dsh-ci-test-reliability/SKILL.md) が割り当て、復元、同期、タイムアウト予算、プラットフォーム、teardown のルールを担い、その[フレーク診断ワークフロー](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/dsh-ci-test-reliability/references/ci-flake-diagnosis.md)が既存の確率的な失敗を分類します。[予期しない fork の終了](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/coverage-fork-diagnostics.ts)は PID、ファイル、コード、シグナルを報告します。

## キー付きの方針: ここでは推論が安い

私たちは DeepSeek です。実 API のテストを節約しないでください。キーのないテストが証明するのは配管であり、エージェントが実際のモデルに対して機能することを証明するのはキー付きの実行だけです。ファイル書き込みのプロンプト、マルチターンの会話、ツール使用、ストリーム途中のキャンセルをカバーします。最も価値が高いのは、出荷済みの `dsh` プロファイルを起動し、プロンプトを 1 つ送り、世界を確認する**スモーク・テスト**です。これらはモックでは捉えられない「ユニット・テストは緑なのに製品が壊れている」類を捉えます（[postmortem 0001](postmortem/0001-acp-default-export-drops-inject.md)）。自己スキップは、秘密のない CI とキーのないコントリビューターを止めないようにするものであり、コストのシグナルではありません。プロファイル・レベルの統合テストは `apps/cli/tests/profiles/` に置き、パッケージ固有のコンポジションはそのパッケージのテストのそばに置きます。

## モックより実際の実装を優先する

高価または非決定的な境界（LLM アダプター、ネットワーク、時計）だけをモックし、それより下流はすべて実際のままにします。手作りの代役が証明するのはブリッジがバイトを運ぶことであり、出荷されるツールが主張どおりに動くことではありません。ブリッジのツール呼び出しテストは、スクリプト化したモック・モデルの背後に実際のツール・レジストリとパイプラインを保ちます。`makeBridgeHarness()` は、`MockAdapter` を唯一のモックとして、ループ、セッション・ストア、ツール・レジストリ、JSONL 永続化をマウントします（packages/acp/acp/tests/harness.ts）。

リカバリのテストは、チャンク前後の失敗を step ごとに分け、失敗したチャンクがメッセージやツールの副作用を導かないことを証明します。枯渇、キャンセル、ポリシーの合成、永続化、ステータス、ワイヤーのカウント、トランスポートを閉じるアイドル・タイムアウト、出荷される Loader のコンポジションをカバーします。

## 自己申告ではなく世界を検証する

e2e のアサーションは、コマンドを再実行するか、ファイルを外部から読み直します。エージェント自身の出力に対するキーワード探索では、ずるをするエージェントが通ってしまいます。触れていないファイルがバイト単位で同一であることをアサートします。e2e テストは自らのリソースを所有します。テスト内で作成し、`afterEach` で dispose し（失敗、リトライ、タイムアウトの場合も）、共有フィクスチャは他の `*.e2e.ts` ではなく素の `tests/harness.ts` に置きます（spec を import すると、その `describe` が再登録され、実 API 呼び出しが重複します）。

## 実際のエントリー・パスをテストする

- 製品から見えるプラグインには、ユニットではない実コンポジションのテストが必要です。手で組み立てた `ctx.plugin(...)` のスイートでは不十分です。テスト専用の `cordis.yml` を Loader とアプリ／プロセスを通じて起動し、外部サービスや非決定的な入力だけをモックし、モデルから見えるリクエスト／ログ、永続状態、またはユーザーから見える出力をアサートします。オプトインは出荷される既定値に入れません。
- ガードは、回帰がそれを失敗させるときにだけガードになります。`inject` を持たないプラグイン（バンドル／コンポジション・プラグイン）では、既定のエクスポートが必要な名前付きエクスポートを置き換えても Loader のスモークは緑のままです。明示的な `expect('default' in mod).toBe(false)` と `unwrapExports` の往復アサーションを追加し、それを証明します。回帰を導入し、赤になるのを見て、元に戻します。
- 「実際のエントリー・パス」とは公開された成果物を意味します。パッケージの `bin` は素の `node` でビルド済みの `lib/bin.js` を実行し、tsx が隠す失敗（settle の競合、モジュール解決、握りつぶされたロード失敗）を露呈します。同じことは、index 以外のランタイム・エントリー（Node プログラムのブートストラップ `lib/process.js`）や、バンドル間で共有されるシングルトン・モジュール（`packages/sdk/server/tests/built-scope-carrier.e2e.ts`）にも当てはまります。ビルド済み成果物のスモークを緑に保ち（`packages/examples/*/tests/built-bin.e2e.ts`、`packages/ptc-runtime/ptc-runtime-node/tests/built-lib.e2e.ts`）、本当に存在しない設定が非ゼロで終了することをアサートします。

## テストの解決: ソース・プレーン限定

- すべての vitest 設定は vite-tsconfig-paths を `tsconfig.base.json` に向け、素のワークスペース import は `src` に解決され（[レイアウト](development.md#typescript-project-layout)）、パッケージの `exports` を通じてビルド済みの `lib/` に解決されることは決してありません。そこにある古い成果物は、モジュール・シングルトンの 2 つ目のコピーをロードします。ビルド済み成果物は明示的にのみ消費されます。`lib` モードのサブプロセスと、後述のビルド済みスモークです。

## テスト用サブプロセスの起動モード

- CI とビルドを持つテスト・レーンは、プロファイルまたは Cordis 設定のサブプロセスをすべて、共有のデュアルモード・ランチャーを通じてビルド済みの `lib/` から実行します。これらのサブプロセスに対して `--import tsx` を手書きしてはいけません。
- Cordis をロードしないプロトコルや OS のフィクスチャは、tsx やルートの paths マップを使わず、消去可能な `.ts` を Node で直接実行します。
- `src` を選択できるのは、ソース・パスの解決自体を主題とするテストだけです。その契約をテスト内で明示します。

## スナップショット・テストが必要なとき

些細でないモデル、プロトコル、人から見える変更はすべて、同じ PR でキーなしの記録済みセッション・シナリオを追加または更新します。パッケージ、e2e、モックのみ、根拠のエビデンスは、組み立て済みのトランスクリプトを置き換えません。Headless、SDK、ACP、Web の記録は `snapshots/session/`、`snapshots/sdk/`、`snapshots/acp/`、`snapshots/web/` に置きます。Web のレンダリングは別のシナリオの正準セッションを明示的に借りることがあります。記録済みセッションに駆動されない期待出力は、所有するアプリ、パッケージ、スクリプトの `tests/expected/` に留め、`*.snapshot.ts` の接尾辞を使いません。[`dsh-session-snapshot`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/test-support/session-snapshot/README.md) が共有の保存ルールとプロファイル・アダプターを担います。Agent loop、セッション・ライフサイクル、`SessionEventMap` の変更は SDK の両方のプロジェクションを更新します。`snapshots/sdk/` が TypeScript を担い、[Python ランタイムの CI](https://github.com/deepseek-ai/deepseek-harness/blob/master/.github/workflows/ci-master.yml) が `scripts/snapshots/python-sdk-single-exe/` を担います。新しいケイパビリティ seam とライフサイクルやトランスクリプトの変種は、計画時に必要なすべての階層を名指しします。
