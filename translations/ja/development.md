# 開発ガイド

日本語 | [English](../../docs/development.md) | [中文](../../docs/development.zh.md)

> これは非公式の日本語訳です。英語版（[docs/development.md](../../docs/development.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

セットアップ・チュートリアルは、新しいコントリビューターを前提条件から検証済みのチェックアウトまで導きます。続くコントリビューター・リファレンスでは、リポジトリのレイアウト、日々のワークフロー、CI の構成を扱います。設計の根拠と実装の詳細は、リンク先の Agent Notes とスクリプトにあります。

## セットアップ・チュートリアル

### 前提条件

- Node.js 24.9 以降。CI は 24.9、最新の 24、26 を対象とします。詳しくは [Node エンジン下限の Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/process/2026-10-15-node-24-runtime-floor.md) を参照してください。
- Node.js の TypeScript 型ストリッピングが有効であること。リポジトリのビルド・スクリプトは tsdown のネイティブ設定ローダーで `tsdown.config.ts` を読み込むため、`NODE_OPTIONS` に `--no-experimental-strip-types` がある場合や、Node.js ビルドが TypeScript のサポートを欠く場合は失敗します。`pnpm run build` はこれを最初に確認し、原因を名指しします。
- Corepack が有効な pnpm。リポジトリは `package.json` で `pnpm@11.28.5` を固定しています。`pnpm --version` が Corepack を通じて解決されない場合は `corepack enable` を実行してください。
- Git 2.26 以降。フックのセットアップが、Git の worktree 固有の設定拡張を有効にします。
- 任意: Web、headless、ACP の自動化デモと実 API の e2e テスト用の DeepSeek API キー。

### Windows と WSL 2

Windows では、ネイティブのツールで開発することも、Linux 環境のために WSL 2 を使うこともできます。WSL 2 は Linux の挙動を検証するときや、ネイティブ依存関係のコンパイルやファイルシステムのパーミッションが Windows での開発を妨げるときに Linux のツールチェーンを使うのに便利です。各環境には独自のランタイム、ビルド・ツール、パーミッションが必要で、WSL は任意です。

チェックアウト、インストール済みの依存関係、ツールチェーンは同じ OS 環境に保ちます。WSL 2 ではチェックアウトを Linux のファイルシステムに置き、ネイティブの Windows ツールでは Windows のファイルシステムを使います。2 つのファイルシステムをまたいだファイル・アクセスは、Git、依存関係のインストール、ビルドのような I/O の多い操作にオーバーヘッドを加えます。Microsoft の[ファイル保存とパフォーマンスのガイダンス](https://learn.microsoft.com/en-us/windows/wsl/filesystems#file-storage-and-performance-across-file-systems)を参照してください。

ネイティブのバイナリやリンクは OS 間で異なりうるため、各環境で依存関係を別々にインストールします。テスト結果はテストを実行した環境に適用され、Windows 固有の挙動には引き続きネイティブの Windows での検証が必要です。

### 初回セットアップ

pnpm は厳密なシンボリックリンクのリンカーを使うため、宣言していない依存関係は hoisting に頼ることなく失敗します。依存関係のビルド・スクリプトは `pnpm-workspace.yaml` で明示的に許可リストに入れてください。

リポジトリのルートから依存関係をインストールします:

```sh
pnpm install
```

インストールは `scripts/install-lefthook.mjs` を通じて worktree ローカルの Lefthook フックも設定します。[worktree ローカルのフックの Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/process/2026-07-27-worktree-local-lefthook.md)が、フック・パスの安全契約を所有します。

[DevTools フロントエンド](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/inspector/README.md#use-this-package)は、固定された npm ソース・パッケージと Vite からローカルでコンパイルされます。依存関係をインストールした後、そのビルドはネットワーク・アクセスもブラウザのインストールも必要としません。ワークスペースと公開パッケージのインストールは DevTools リソースのダウンロード・フックを実行せず、公開された Inspector パッケージにはビルド済みのフロントエンドが含まれます。

依存関係がキャッシュから復元された、または `postinstall` がスキップされたためにフックがない場合は、手動でインストールします:

```sh
node scripts/install-lefthook.mjs
```

ラッパーが既存の Git 設定を拒否する、または古いロックを報告する場合は、worktree のメタデータを推測で編集せず、その診断とリンク先の Agent Note に従ってください。チェックアウトを移動した後は、ラッパーを再実行して所有パスを再生成します。

新しいクローンの後に 1 回 typecheck を実行します:

```sh
pnpm run typecheck
```

`pnpm run typecheck` が正常に終了すれば、セットアップは完了です。

## コントリビューター・リファレンス

### TypeScript プロジェクトのレイアウト

リポジトリは分離された Host と Client のアグリゲートを使います。通常のパッケージはちょうど 1 つのアグリゲートに登録されます。Host のパッケージは `tsconfig.host.json` に、Client のパッケージは `tsconfig.client.json` に登録します。3 つのパッケージ（`host/webserver`、`compaction/compaction`、`typert/registry`）は共有リーフとして両方のアグリゲートから参照され、どちらの側も同じソースを型チェックします。

| ファイル | 役割 | プログラムを形成するか |
|---|---|---|
| `tsconfig.json` | ソリューションのルート: ベース設定を `extends` し、`files: []` と 2 つのアグリゲートへの参照を持つ。tsserver の発見エントリーであり、完全な Project Reference グラフを明示的に実行する際のエントリーでもある。継承した `paths` を通じて、`scripts/` を実行する tsx の解決設定にもなる | いいえ |
| `tsconfig.host.json` | Host アグリゲート: Host のパッケージ、examples、tests、scripts、website、および `api/remotes` の例外的な Host プロジェクト | はい |
| `tsconfig.client.json` | Client アグリゲート: `packages/client/*` のパッケージとそのテスト、`apps/web`、および `api/remotes` の例外的な Client プロジェクト | はい |
| `tsconfig.base.json` | 共有の compilerOptions とソースの `paths` マップ。vitest の設定が vite-tsconfig-paths を向ける解決のファサードでもある。`include` を持たないため、その `paths` はすべての importer に適用される | いいえ |
| `tsconfig.base.client.json` | Client アグリゲートとすべての `packages/client/*` パッケージが拡張する、ブラウザ向けのコンパイラー設定（`jsx`、DOM ライブラリ、`types: []`） | いいえ |

Host と Client のアグリゲートの型チェックは、8 GiB の Node old-space 制限で実行されます。これはコンパイラーごとの制限であり、並行するビルド・ステップには追加のメモリが必要です。`pnpm run build:official --serial` は、`DSH_GATE_CONCURRENCY` が設定されていてもビルド・ステップの重複を防ぎます。DSH の公開ワークフローはこのオプションを使います。

Host と Client が 2 つのアグリゲート・プログラムのままであるのは、両側が cordis の `Context` インターフェースを、同じキーで異なるサービスとともに宣言マージするためです。1 つのプログラムが両方のマージを見ると衝突を報告します。衝突は `ts.Program` の内側にのみ存在し、モジュール解決がそれを引き起こすことはありません。そのためソリューションは両方のアグリゲートを参照でき、1 つの paths ファサードが両側にまたがれます。そこから 3 つの規律が導かれます:

- `tsconfig.base.json` は決して `include` や `files` を持たない。それらは拡張するすべてのパッケージ・プロジェクトに漏れ、ファサードの全一致スコープを狭めてしまう。
- リポジトリ全体の `ts.Program` を構築するスクリプトは、`tsconfig.host.json` か `tsconfig.client.json` を明示的にシードする。ルートのソリューションは決して使わない。両方のアグリゲートを 1 つのプログラムに平坦化すると、`Context` のマージが衝突するためである。
- 新しいパッケージはちょうど 1 つのアグリゲートに登録する。両方のリーフ設定を持つのは上記の分割パッケージだけで、共有リーフは、どちらの側も同じソースを型チェックしなければならないため両方のアグリゲートに登録する。Node ローダーのエントリーとブラウザのエントリーの両方を持つことは、パッケージを分割する理由にならない。通常の Client プラグインは、Client のビルド・フェイズで両方のランタイム成果物を生成する。

7 つのパッケージが Host と Client の tsconfig を分割します。`api/remotes`、`api/gateway`、`api/session-controller`、`api/workspace-controller`、`client/connection`、`session-query/session-log-export`、`experimental/liveshare-protocol` です。`api/remotes` の Host エントリーは Host の Typert グラフに参加し、Client エントリーは生成された `/remote` 宣言を import します。`session-log-export` は Node のアーカイブ生成をブラウザ・コントローラーから遠ざけ、`liveshare-protocol` は Node のワイヤー codec を Host のリーフに、ブラウザ・ビューアーのキャリアを Client のリーフに保ちます。したがって分割パッケージのルートの `tsconfig.json` はいずれも単なるソリューションであり、2 つのアグリゲートと直接の利用者はそれぞれ `tsconfig.host.json` または `tsconfig.client.json` を参照します。ワークスペースの `constraints` ゲートは到達可能な Project Reference グラフをたどり、参照する各プロジェクト自身のコンパイラー・フェイスを検査します。単一設定のターゲットはどちらのフェイスからも有効なままで、分割ターゲットはソリューションのルートや反対側のリーフではなく、一致するリーフを名指さなければなりません。分割パッケージは両方のリーフ設定の存在から発見されるため、新しい分割は自動的にゲートに加わります。[`api-remotes` の README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/remotes/README.md)と[`session-log-export` の README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/session-query/session-log-export/README.md)が、それぞれの分割を説明しています。

`pnpm run build:artifacts` はすべての成果物を出力し、各コンパイラー・フェイスのパッケージ参照だけをチェックしてコンパイルします。Client のコンパイルは Host と Desktop のバンドルが終わるまで待ちます。分割されたコンパイラー・フェイスが、それらのバンドルが読む共有の `lib/types` ファイルを書き換えるためです。Client のバンドルは Client のコンパイルに続きます。成果物の順序は次のとおりです:

```sh
pnpm run build:native-system
pnpm exec tsx scripts/generate-typert.ts
tsc -b <tsconfig.host.json references>
tsdown --env.DSH_BUILD_FACE host
pnpm --filter @deepseek-ai/dsh-desktop run bundle
tsc -b <tsconfig.client.json references>
tsdown --env.DSH_BUILD_FACE client
pnpm run build:web
```

`pnpm run build:check` は `tsc -p tsconfig.host.json` と `tsc -p tsconfig.client.json` だけを並行して実行します。プロジェクト・モードは、各アグリゲートのリポジトリ全体のテストとスクリプトのプログラムを、すでにディスク上にあるパッケージ宣言と照らしてチェックし、参照をコンパイルすることはありません。そのため `build:check` は、欠落した、または古い `build:artifacts` の出力を検出も再ビルドもしません。欠落した出力はコンパイラー・エラーとして現れます。`pnpm run build` は両方のステップ群を 1 つの[スケジュールされた](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/run-gates.ts)依存関係グラフとして実行します。各アグリゲートのチェックは、自身のフェイスのパッケージのコンパイルが終わると開始し、残りの成果物ステップと重なります。3 つのコマンドすべてで、`DSH_GATE_CONCURRENCY` が並行するステップを制限し（`1` は依存順に実行）、`DSH_GATE_FAIL_FAST=1` は最初の失敗で残りのステップを停止します。そうでなければ失敗は依存するステップだけをスキップするため、独立したエラーがすべて報告されます。失敗したステップは自身の出力を表示し、どの失敗も非ゼロで終了し、client のビルド記録を書きません。

変更に何が必要かでコマンドを選びます:

| 状況 | コマンド |
|---|---|
| パッケージのソースを反復し、実行可能な成果物が必要 | `pnpm run build:artifacts`。`dev:web` と `dev:desktop` は起動前にこれを実行する |
| テスト、スクリプト、ベンチマーク、website のコードを型チェック | 最新の `build:artifacts` の後に `pnpm run build:check`。パッケージのソースを変えた場合は先に `build:artifacts` を再実行する |
| TypeScript の作業を完了と宣言する前 | 最新の `build:artifacts` の後に `pnpm run build:check`、または `pnpm run build` |
| 完全な検証。必須の CI ビルド・レーンが実行する形 | `pnpm run build` |
| Web と Desktop の成果物なしの型チェック | `pnpm run typecheck` |
| Lint またはドキュメントのコード・ブロックのチェック | `pnpm run lint` または `pnpm run doc-typecheck`。どちらも `host-contracts` を自身で準備する |

PR の成果物ビルダーは `build:artifacts` を実行します。必須の Linux と Windows の通常ビルド、公開の `typecheck` は、リポジトリ全体のテストとスクリプトのプログラムもチェックします。CI のベンチマーク準備はさらに、使い捨てのライブラリ出力に `--noCheck` を使い、診断はそれらの必須ビルドに頼ります。他の成果物ビルダーはパッケージのチェックを保ちます（[根拠](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/process/2026-10-05-pr-artifact-typechecks.md)）。

両方の tsdown パスは `vendor/*`、`packages/*/*`、`apps/cli` に一致し、Host のパスは `apps/desktop-host` にも一致します。どちらもビルド成果物を走査して Client パッケージを発見せず、Host／Client のパッケージ・フィルター・リストも維持しません。パッケージ・ローカルの tsdown 設定は、`DSH_BUILD_FACE` を通じて現在のフェイズのエントリーを選びます。通常の Client プラグインは Client フェイズに Node ローダーとブラウザ・バンドルの両方を生成し、`api-remotes` は `hostPhase: true` を使って Host のエントリーを早めに生成し、Client フェイズにはブラウザ・バンドルだけを残します。Tsdown は、先行する tsc フェイズが `lib/types` に出力した JavaScript だけを消費します。Tsdown は一致したワークスペース・メンバーを並行してビルドするため、メインのバンドルがワークスペースの devDependencies をその `lib/` 出力からインライン化する `apps/desktop` は、Host パスの後に独自のステップでバンドルします（[Desktop の README](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/desktop/README.md#bundled-workspace-dependencies)）。

Typert は `tsconfig.host.json` を通じて Host の型を解析し、パッケージのコンパイル前に Host のリフレクションと Host-for-Client の Remote 宣言を生成します。ソースのみの [`generate-typert.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/generate-typert.ts) パスは、ビルド済みの出力を必要とするワークスペースの import を拒否します。生成された宣言に戻ってくるメタデータの依存関係も含みます。Host の tsdown は、メタデータを再度生成せずにソースのデコレーターを下げ、Client の tsdown も既存のメタデータを消費します。`pnpm run typecheck` はメタデータを生成し、Host と Client のパッケージを参照順にコンパイルし、各アグリゲートをパッケージのコンパイル後に `tsc -b` でチェックします。ネイティブ・アドオン、バンドル、Web、client のビルド記録はスキップします。`lint`、`lint:fix`、`doc-typecheck` は `compile-referenced-projects.ts host-contracts` を通じてメタデータと Host のパッケージ宣言を準備します。生成の失敗は依存するコンパイルを停止します。これらの宣言のみのコマンドはランタイムのバンドルを更新しません。ビルド済みの出力を起動またはテストする前に `build:artifacts` を実行してください。

`pnpm run build` と `build:artifacts` は、ルート・パッケージのバージョン、7 文字のソース・コミット、Git がローカルの変更を報告する場合の dirty マーカーを埋め込みます。呼び出し元が指定した他の `DSH_CLIENT_*` の値も継承します。`pnpm run build:official` は CI とリリースの成果物ビルドのクロスプラットフォームなローカル等価物で、ローカルの dirty マーカーを省略します。成功した各 `build` または `build:artifacts` は、正確な公開値を Vite の出力と動的な client バンドルに結び付ける gitignore 済みの記録を書き、リリースのパッキングとビルド済み Web のテストは、記録の欠落や、後の部分ビルドで変わった成果物を拒否します。`pnpm run dev:web` は最初に `build:artifacts` を実行し（`--skip-build` は既存の成果物ツリーを再利用します）、その後、現在のバージョンと Git の状態を 1 回サンプリングし、その環境をセッションのすべてのウォッチャー・ステージで共有します。ウォッチャー・ステージが記録済みの成果物を書き換えるため、完全ビルドの記録は検証しません。

`pnpm run build --profile local` はローカルの client ブランディングを明示的に選び、リポジトリのバージョン、コミット、dirty のメタデータを保ち、継承した client のタイトルその他の公開オーバーライドを破棄します。`build:official` は正確な公式の値を保ちます。[Desktop のパッケージング](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/desktop/packaging/README.md#configuration-and-local-builds)は、署名モードごとにプロファイルを選びます。

静的解析とテストは、ベースの `paths` マップを通じてワークスペースの import を `src` に解決し、クリーンなツリーで通らなければなりません。ビルド済みの `lib/` 出力を消費するゲートは、その依存関係を明示的に宣言します。生成された Host-for-Client の Remote 宣言は意図的な例外です。公開の `typecheck`、`lint`、`doc-typecheck` のコマンドはそれを最初に生成し、内部の `*:contracts-ready` スクリプトは、呼び出し元の公開コマンドやスケジューラー・ゲートが Typert の契約生成パスか完全なビルドにすでに依存していると仮定します。ゲート準備の契約は [Typert Remote のノート](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-02-typert-remote-method-calls.md)を参照してください。

ビジネス・サービスは Host 上で呼び出し可能なメソッドを `@Remote` または `@RemoteScope` で宣言します。Host のビルドは Host-for-Client の型とランタイムのコントリビューションを生成し、Client の `api-remotes` のコンポジションがそれらのコントリビューションを `ctx.remote` とスコープ付きの `agentCtx.remote` 名前空間の下でロードします。両側の生成物、それらのアセンブリ関係、SRC の開発フォールバック、Web のビルド順序は [API ゲートウェイ](api-gateway.md)を参照してください。

関連するローカルのチェックがビルド済みのパッケージ出力を消費する場合は、先に 1 回ビルドします:

```sh
pnpm run build
```

`pnpm run hygiene` は、ビルド済みの `lib/*.js` ファイルに対してパッケージのエントリーポイントを検証する `publint` と、ビルド済みの宣言を一時的な NodeNext の利用者に対して検証する `verify-node-next-types` を含みます。新しい worktree には、`pnpm run build` が実行されるまでバンドル済みの JS も宣言もありません。通常のコミットと push は、選択したチェックがそれを消費しない限り、そのビルドを必要としません。

### 環境変数

実際の DeepSeek アダプターとキー付きの agent デモは、環境またはリポジトリ・ルートの gitignore 済み `.env` から認証情報を読みます:

```sh
DEEPSEEK_API_KEY=sk-...
DEEPSEEK_BASE_URL=https://... # optional
```

`DEEPSEEK_BASE_URL` は任意で、既定は公開 API です。実際の認証情報をコミットしてはいけません。実 API の e2e スイートは、`DEEPSEEK_API_KEY` が設定されていない場合に自己スキップします。

### Git 統合

`.i18n.yaml` の記録は Git の既定のテキスト・マージを使います。記録が衝突するのは、両方のブランチが同じ見出しセクションで言語固有の内容を変えた場合だけです。Markdown を解決してから `pnpm run verify-translation-pairing --write <pair>` を再実行します。`pre-merge-commit` がそれ以外はクリーンなマージを拒否した場合、Git はコミットなしで完全な結果をステージしたままにします。失敗を修復して `git commit` を実行するか、`git merge --abort` を実行します。

lefthook は高速なローカル・チェックポイントとして `lefthook.yml` に設定されています:

- `pre-commit` は、ステージされたペアリング記録をステージされた owner の blob と照らして検証し、プロジェクト非依存の `.oxlintrc.staged.json` プロファイルでステージされたファイルを検証して Oxlint の修正を 1 回の制限付きリトライで適用し、ステージされた diff の空白エラーをチェックし、pi-ai プロバイダーのメタデータがその入力や生成ファイルとともにステージされている場合にチェックし、vendor manifest のガードを実行します。
- `pre-merge-commit` は、Git が自動マージ・コミットを作成する前に、同じインデックスに基づくペアリング・チェックを実行します。
- `pre-push` は `pnpm run typecheck` を実行し、Client の TypeScript チェックの前にバンドルなしで Typert の契約を生成し、両方のアグリゲートをチェックします。

vendor manifest のガードは、`vendor/*/src` 以下の変更が、対応する `vendor/README.md` の manifest 更新とともにステージされていることを検査します。vendored なコードを編集する前に `vendor/README.md` を参照してください。

pi-ai のチェックは、期待されるメタデータをメモリ内で再生成します。古いデータを、ファイルを書き換えたりステージしたりせずに拒否します。pi-ai の依存関係や生成器を変えた後は、そのパッケージの[プロバイダー・メタデータの更新手順](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/llm-pi-ai/README.md#generated-provider-metadata)に従ってください。

スコープ付きのペアリングとプロバイダー・メタデータのチェックを除き、フックは意図的にテスト、スナップショット、ドキュメントのチェック、ビルド、hygiene を実行しません。コントリビューターは[変更した挙動に関連するチェック](../../AGENTS.md#run-relevant-checks-locally)を 1 回実行します。CI が網羅的なカバレッジ、ビルド済み成果物のスモーク、Node 24.9、最新の 24、26 の互換性マトリクスを所有します。

コントリビューターは `pnpm run check:all` で包括的なローカル・ゲート群にオプトインできます。このコマンドは Git のフックから独立しており、agent への指示ではありません。

CI の外（`CI` が `true` 以外）では、既定のワーカー数は[ローカルのワーカー予算](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/parallelism.ts)に従います。ゲート・スケジューラー、`build`、`typecheck`、Oxlint、publint は 12、Vitest の fork とローカルのカバレッジは 48 です。`doc-sync`、`test:docs`、`hygiene`、`check:all` は CI でも最大 8 個のゲートを同時に実行します。`os.availableParallelism()` はコンテナの CPU クォータではなくホストの CPU を報告するため、上限のない既定値は共有ホストを過剰に購読させます。CI はランナーの CPU 数を保ちます。`DSH_GATE_CONCURRENCY`、`DSH_OXLINT_THREADS`、`DSH_COVERAGE_MAX_WORKERS`、Vitest の `--maxWorkers` が既定値を上書きします。

### CI ゲート

キーのない [CI ワークフロー](https://github.com/deepseek-ai/deepseek-harness/blob/master/.github/workflows/ci.yml)は、独立したゲートを広いレーンにまとめ、サポートされる Node バージョンにわたってより小さな互換性シグナルを実行します。成果物の利用者は、そのレーン内の 1 つのビルドを待ちます。カバレッジのパーティションは、自分のプラットフォームとランナー・プールにスコープされたファイルのタイミングを再利用し、設定されたワーカー上限の中で作業を分散します。ネイティブの Windows レーンは別の [CI Windows ワークフロー](https://github.com/deepseek-ai/deepseek-harness/blob/master/.github/workflows/ci-windows.yml)で、Python のチェックは別の [CI Python ワークフロー](https://github.com/deepseek-ai/deepseek-harness/blob/master/.github/workflows/ci-python.yml)で実行され、それぞれ独立に再実行できます。必須のベンチマークは、標準の GitHub ホストの Linux で別々に実行されます。ルーティングとジョブのタイムアウトは[ベンチマーク・ランナーのリファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/benchmarks/AGENTS.md)が所有します。別の実 API ワークフローは、設定されたワーカー上限で `pnpm run test:e2e` を実行します。現在のゲートとジョブの目録は [scripts/run-gates.ts](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/run-gates.ts) とワークフロー・ファイルを参照してください。

認証情報を必要としない dsh の依存関係レイアウトと dsh/vendor のパック・リハーサルは、`DSH_CI_FAILOVER_LINUX=selfhosted` で、イベントが信頼された master への push または同一リポジトリの非 fork、非 Dependabot の pull request である場合にのみ、既存の Linux セルフホスト・プールを使います。手動のディスパッチを含むその他のすべての場合は `ubuntu-24.94` を使い、手動の公開はホストされたままとします。永続ストアの分離とフォールバックの制限は[リリース・リハーサルのランナーのリファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/process/2026-07-26-ci-failover-runbook.md)を参照してください。

### 日々のコマンド

ルートの[コントリビューター向けの指示](../../AGENTS.md#commands)が一般的なコマンドを要約し、現在のスクリプトとゲートの目録は [`package.json`](https://github.com/deepseek-ai/deepseek-harness/blob/master/package.json) と [scripts/run-gates.ts](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/run-gates.ts) が所有します。変更したサーフェスをカバーする最小のチェックを選びます。ドキュメントの変更は `pnpm run doc-sync` を使います。パッケージの公開挙動の変更は、所有する README や JSDoc も更新し、ビルド済み成果物のチェックには先に `pnpm run build` が必要です。

### プロファイル実行

これらのソース・チェックアウトのデモを使う前に、リポジトリのビルドを別途実行します:

```sh
pnpm run build
```

ワンショットの Headless コーディング agent には、環境またはリポジトリ・ルートの `.env` にある `DEEPSEEK_API_KEY` が必要です:

```sh
pnpm dsh --profile headless "summarize this workspace"
```

PTC モードのデモは、コード・プレゼンテーションを有効にして同じ headless プロファイルを実行します:

```sh
pnpm run demo:ptc -- "summarize this workspace"
```

### アプリケーション・コマンド

Web と Desktop は 1 組のコマンドを共有します。`start:*` は先行する `pnpm run build` または `build:artifacts` の成果物を起動し、`dev:*` は最初に `build:artifacts` を実行してから起動します。Web はさらに、ソースの編集で client バンドルを再ビルドし続けます。Host はソースから実行され、ブラウザはビルド済みのバンドルをロードするためです:

```sh
pnpm run start:web       # serve built Web artifacts through the source launcher (the same launch as pnpm dsh web)
pnpm run dev:web         # build, serve, and rebuild Web client bundles on source edits
pnpm run start:desktop   # launch built Desktop artifacts
pnpm run dev:desktop     # build, then launch Desktop
```

Web コマンドの後の引数は `dsh web` に届きます。たとえば `pnpm run dev:web --no-open --port 3081` です。`dev:web` は `--skip-build` も受け付け、既存の成果物ツリーを再利用します。`--no-serve` は、別の場所で起動したサーバーのそばで再ビルドのウォッチャーだけを実行します。両方の Web コマンドは通常の Harness home を使い、Desktop のコマンドは [Desktop の README](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/desktop/README.md)で説明されている隔離された開発用 home を使います。ルートの `Makefile` は同じコマンドを `make web`、`make dev-web`、`make desktop`、`make dev-desktop`、`make build` として名指しし、`ARGS='--no-open'` がオプションを転送します。

### TODO マーカー

コード内の既知の問題には、緊急度の順に 3 つのコメント・タグのいずれかを使います:

- `FIXME` — 新しいリリースを妨げるべき問題。レビュアーが明示的に、その変更をそのままマージしてよいと合意しない限り、未解決の `FIXME` を抱えてリリースすべきではありません。
- `TODO` — リソースが確保でき次第、早めに修正すべき問題。
- `XXX` — いつか修正するかもしれない問題。優先度は最も低く、約束はありません。

コードを走査する人がリリースの妨げと「いつかやるかも」を見分けられるよう、緊急度に合うタグを選びます。

### 型をそのまま文書化する（`ts type-equiv`）

[subsystems](subsystems/index.md) のページは、ソースと等価な宣言を元の JSDoc とともに貼り付け、読者が正確な型定義とソースの契約を見られるようにします。ソースの変更で貼り付けがドリフトしないよう、` ```ts ` の代わりに ` ```ts type-equiv ` としてフェンスし、`scripts/type-equiv.manifest.json` に、写し元のソース・ファイルとシンボルを登録します:

```json
{ "doc": "docs/subsystems/session.md", "symbol": "SessionEvent", "source": "packages/core/session/src/types.ts" }
```

その後 `pnpm run verify-type-equiv`（`doc-sync` の一部）が、TypeScript パーサーを通じてソースからそのシンボルの宣言と付随する JSDoc を抽出し、ブロックが両方に一致することをアサートします。実装本体がカタログに属さないクラスには ` ```ts public-api ` を使い、`"projection": "public-api"` を設定します。チェックされるプロジェクションは、公開フィールド、コンストラクター、アクセサー、メソッド、元のクラス／メンバーの JSDoc を保ち、本体と private または protected のメンバーを省略します。比較は空白と JSDoc 以外のコメントを無視しますが、メンバーのドキュメントを含むすべての元の JSDoc コメントを要求するため、読者は正確な型定義のそばでソースの契約を見られます。ゲートは、プライマリ・ブロックと manifest のエントリーの間に、ドキュメント、シンボル、プロジェクションごとの 1:1 の対応を強制します。対になる `.zh.md` のブロックは、追跡されるフェンス列全体がバイト単位で同一で、同じ順序である場合にのみ、接尾辞のない兄弟のエントリーを再利用します。`doc-typecheck` はコンパイル可能なフェンスに同じ派生のルールを適用し、両方のソース等価フェンスをコンパイルとそのオプトアウト比率から除外します。ドキュメント化された宣言やその JSDoc を変えると、貼り付けを更新するまでゲートは失敗します。プライマリ・ブロックを追加または削除するときは、同じ変更で manifest を更新します。
