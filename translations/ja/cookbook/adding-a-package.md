# クックブック: workspace パッケージの追加

日本語 | [English](../../../docs/cookbook/adding-a-package.md) | [中文](../../../docs/cookbook/adding-a-package.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cookbook/adding-a-package.md](../../../docs/cookbook/adding-a-package.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

新しい `@deepseek-ai/dsh-<name>` パッケージのためのファイル単位のチェックリストです。このチェックリストは bash パッケージと adapter パッケージをテンプレートとして検証されています。それらと乖離した場合は、ここを修正してください。

## 1. パッケージを作成する

```
packages/<group>/<pkg>/
  package.json     # copy from packages/core/tools, adjust name/description/deps
  tsconfig.json    # extends ../../../tsconfig.base.json, rootDir src,
                   # outDir lib/types, references: ../../../vendor/cosmokit,
                   # ../../../vendor/cordis (+ ../../../vendor/schemastery if
                   # you use Config, + ../../<group>/<dep> for each dsh dep)
  src/index.ts     # service default export or plugin (name/inject/apply/Config)
  locale/en.json   # optional display metadata: meta.title, meta.description, and meta.detail
  locale/zh.json   # translations using the same fields
  README.md        # service API, events, extension points, design notes,
                   # + gated Model Experience context blocks or short form
                   # + the gated "Known Limitations and Deferred Work" section
                   # (or a whitelist entry in scripts/verify-package-readme-limitations.ts)
```

パッケージの役割に合う既存グループがあれば、それを選びます（`core`、`llm`、`shell`、`compaction`、`subagent`、`todo`、`session`、`client`/`host`、`util`、`test-support`）。新しいグループも許されますが、それは純粋なコンテナです。`package.json` もソースファイルも持たず、パッケージは依然としてそのちょうど1階層下に置かれます。

package.json の不変条件（`pnpm run constraints` / `scripts/check-workspace-constraints.ts` が強制）: `private: true`、ルートの `package.json` と一致する `version`、`type: module`、`main: "lib/index.js"`、`types: "lib/types/index.d.ts"`、`exports["."].types: "./lib/types/index.d.ts"`、`exports["."].default: "./lib/index.js"`、`@deepseek-ai/cordis` を peerDependencies と devDependencies の両方に（同じレンジで）置くこと。すべての dsh peer 依存を devDependencies にも反映します。`@deepseek-ai/schemastery` は `dependencies` に入れ（実行時バリデータであり）、agent-loop に合わせます。`files` リストには `lib/index.js`、`lib/types/**/*.d.ts`、ゲートが認識するパッケージ固有の実行時アーティファクトだけを含めます。実行時の export が出力ツリーを指すパッケージは、`lib/types/**/*.js` も含めます。`src`、declaration map、JS map、古いルート宣言ファイルを公開してはいけません。パッケージの `bin` を持つ CLI アプリ・パッケージは、`files` 内の `lib/index.js` の直後に `lib/bin.js` を含めます。

パッケージ内の相対 import は、ソース内で明示的な `.ts` 指定子を使います（例: `export * from './types.ts'`）。コンパイラはそれらを出力 JS では `.js` に書き換え、宣言ファイルには明示的な `.ts` 指定子を残します。標準的な NodeNext/Node16 の TypeScript コンシューマーは、それを隣接する `.d.ts` ファイルに解決します。

## 2. ルート設定に登録する

| ファイル | 変更 |
|---|---|
| `tsconfig.base.json` | 既存グループなら編集不要。新しいグループなら、`@deepseek-ai/dsh-*` ワイルドカードに `./packages/<group>/*/src` 候補を追加する |
| `tsconfig.host.json`（Host パッケージ）または `tsconfig.client.json`（Client パッケージ） | `references` に `{ "path": "./packages/<group>/<pkg>" }` を追加する — 通常のパッケージはちょうど1つの集約に属し、両方には属さない。`api/remotes` はリポジトリ固有の分割を使う。Host が生成したコントラクトを Client が後のフェーズで消費するためであり、新しいパッケージはそれをコピーしてはならない（[レイアウト](../development.md#typescript-project-layout)） |

`packages/client/*` パッケージはさらに `tsconfig.base.json` の代わりに `tsconfig.base.client.json` を拡張し、client プラグイン・パッケージは package.json で `dsh.client` を宣言し、`./client` を export し、共有の tsdown プリセット（`packages/client/tsdown.client.ts`）を呼びます — client 側のコントラクトは [packages/client/AGENTS.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/AGENTS.md) を参照してください。

glob またはパッケージ・マニフェストの検出によって自動的にカバーされ、編集は不要です: ルート `package.json` の workspaces、`scripts/publint-all.ts`、`tsdown.config.ts`、`.oxlintrc.json`、`scripts/check-workspace-constraints.ts`。

## 3. パッケージのトポロジーを決める

交換可能なケイパビリティでは、Service Definition / Service Provider / Consumer の役割を、それぞれが独立して進化する場合に別パッケージへ分けます（docs/architecture.md の「Capability seams」を参照 — shell の3つ組がテンプレートです）。単一目的のプラグインは1パッケージのままにします。

### 存在する役割に名前を付ける

安定した現在の責務に名前を付けます。最初の実装、将来ありうる拡張、Cordis の基底クラスに名前を付けてはいけません。インターフェース・パッケージはケイパビリティに名前を付けます。実装パッケージは、それを区別するメカニズム、プロトコル、環境、ベンダーを加えます。`local` は、同一ホストでの実行がコントラクトの一部である場合にのみ使います。

1つのエンジン、ランタイム、ポリシー、コントローラー、リゾルバー、ストア、現在の設定には単数の `ctx` キーを使います。レジストリや、複数の名前付きメンバーを所有するサービスには複数形のキーを使います。クラスの役割とキーの数は一致していなければなりません。互換性のない host と client の宣言に1つの Cordis `Context` キーを再利用してはいけません。TypeScript の宣言マージは、別々のランタイム・コンテキストを使っていても両方の面を見ます。自然な複数形がすでに別の面に属する場合は、役割の接尾辞を加えます。

| 語 | 使う場合 | 使わない場合 |
|---|---|---|
| `Controller` | コマンドやユーザーの意図を受け取り、既存の1つのドメイン状態または表示状態を変更する。 | 任意の作業を実行する、プロバイダーの一群を所有する、表示のためだけに値を変換する。 |
| `Store` | 1つのデータセットを所有し、主にそのデータの CRUD、スナップショット、購読操作を提供する。 | 状態機械を検証する、権限を仲裁する、作業をディスパッチする、プロバイダーの優先順位を所有する。マップであるからといって、そのクラスがストアになるわけではない。 |
| `Directory` | 発見や選択のためにエントリーとメタデータを公開する。 | プロデューサーが任意の実装を登録する、呼び出し元がそれを介して作業を実行する。 |
| `Presenter` | ドメイン値やツール引数から描画意図への純粋な変換である。 | I/O を行う、購読する、状態を変更する、ライフサイクルを所有する。 |
| `Registry` | 名前付き登録の動的な集合を所有する。lookup、重複や優先順位のルール、寿命、破棄を含む。 | 主なコントラクトがディスパッチ、実行、キャンセル、ポリシー、オーケストレーションである。 |
| `Runtime` | ライブな作業を実行し、呼び出しをまたぐディスパッチ、キャンセル、プロバイダー調整、操作ライフサイクルを所有する。 | レコードを保存するだけ、カタログを返すだけ、1つの値を解決するだけ、設定を保持するだけである。 |
| `Resolver` | 与えられた入力から1つの答えを計算または特定し、その答えのライフサイクルは所有しない。 | 可変のコレクションや長時間の実行を所有する。 |
| `Binder` | 宣言された1つのインターフェースを呼び出し元のコンテキストやライフサイクルに接続し、束縛された値を返す。 | 値をコレクションとして所有する、そのドメイン状態を制御する、データを変換するだけである。 |
| `Engine` | ドメインのアルゴリズムや状態を持つ実行モデルを実装する。 | プロバイダーを選択するだけ、プロトコル境界を越えて転送するだけである。 |
| `Policy` | 何が許可され、選択され、制限され、観察されるかを決める。 | その決定が許可するメカニズムを実行する。 |
| `Executor` | 1つの明示的なリクエストまたは解決済み仕様を、1つのケイパビリティ内で実行する。 | 広範なアプリケーション・ライフサイクルやプロバイダー・カタログを所有する。 |
| `Gateway` | プロセス、ネットワーク、RPC、API の境界を適応する。 | 同一プロセスのサービスを登録するだけ、メタデータを保存するだけである。 |
| `Provider` | ケイパビリティ定義の1つの実装を供給する。複数存在しうる場合はメカニズムやベンダーの修飾語を加える。 | ケイパビリティ定義、プロバイダー・レジストリ、コンシューマー・ランタイムである。 |
| `Backend` | 定義されたインターフェースの背後で、交換可能な下位の永続化、トランスポート、実行を実装する。 | ユーザー向けサービスである、返された1つのライブ・リソース参照である。 |
| `Handle` | 1つのライブ・リソースを参照し、そのリソースを制御または観察する。 | リソース・プール全体を作成・管理する。 |
| `Config` | 1つの解決済み設定値、または厳密に境界付けられた1つのレコードとその更新コントラクトを所有する。 | 一般的なコレクションを保存する、作業を実行する、無関係な設定を公開する。 |
| `Service` | 上のどのより具体的な役割も正直に言い表せない、まとまりのあるドメイン・サービスを所有する。 | クラスが Cordis の `Service` を継承しているという理由だけで名前が存在する。 |

`SDK` は、サポートされている Python および TypeScript の SDK が使う JSON-RPC クライアント／サーバー・プロトコルにのみ使います。DeepSeek Harness 自体は agent harness であり、SDK プロジェクトではありません。製品の正規表記 `Typert` を使い、`TypeRT` や `typeRT` は決して使いません。

<a id="4-write-the-package-readme"></a>

## 4. パッケージ README を書く

パッケージ固有のサービス API、設定、イベント、拡張ポイント、設計メモを最初に置きます。[dsh-doc メタデータ・リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/dsh-doc/references/metadata-links-i18n.md#the-kind-system) の4つの kind ラベル — group、reference、library、bundle — から frontmatter の `kind` を選び、パッケージのリポジトリ上の位置とエントリーの形に合わせます。各 kind は1つの README テンプレートを選びます。制限事項セクションには、このパッケージが所有する永続的なコンシューマー・ギャップと、自明でないメンテナー制約を記録します。通常のクリーンアップは、そのソースの TODO または Agent Note に残します。間接的な Model Experience の文は、このパッケージの貢献を表面化するコンシューマーに言及してもかまいませんが、そのコンシューマーの実装を繰り返してはいけません。パッケージ README は、次の正規の並びで終えます:

````markdown
## Model Experience

### Request context and condition

#### What the model sees

The exact data-dependent fields, an anchored generated-catalog link, or an introduction to the verbatim literal below.

##### Verbatim text for this field, when needed

```markdown
Stable system-prompt prose of any length, or another long non-generated literal, copied exactly from source.
```

#### Token effect

Fixed, conditional, retained, replaced, capped, or zero-direct token effect.

#### KV Cache effect

Append-only, prefix-stable, replacing, or independent behavior, including the exact conditions that may invalidate reuse.

## Known Limitations and Deferred Work

- **Consumer-visible gap** — exact missing operation or case, its consequence, and any maintainer constraint.
````

Model Experience は実装から記入します。直接的、条件付き、上限付き、ライフタイム、補助的なモデル・コンテキストのエントリーごとに1つの H3 を使い、上に示した3つの順序付き H4 フィールドと、それぞれの下に1つの散文段落を置きます。パッケージが所有する安定したテキストを引用します。システム・プロンプトの散文は、それを導入するフィールド（通常は `What the model sees`）の下に、タイトル付き H5 と `markdown` フェンスを置きます。その他の短いリテラルは名前付きプレースホルダーとともにインラインに留め、その他の長いリテラルは同じネスト形式を使います。要約するのは、データ依存のテキストまたはプロバイダー所有のテキストだけです。ツール・スキーマのエントリーは、生成された [ツール・カタログ](../../../docs/tool-catalog.md) のアンカー付きセクションにリンクし、そこにない差分だけを記述します。スコープ設定によって一方だけが隠れうる場合は、プロンプトとスキーマのエントリーを分けてください。`KV Cache effect` では、追記のみの成長、安定して繰り返されるプレフィックス、以前のリクエスト・トークンの置換、独立したモデル・リクエストを区別し、再利用を無効にしうるパッケージ所有の変更を名指しします。「無効にしない」とは、パッケージがすでに再利用可能なプレフィックスを保持することを意味します。プロバイダーのキャッシュ可用性とエビクションはパッケージのコントラクト外のままです。[散文標準](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/dsh-prose-standard/SKILL.md)が完全性と所有権を定め、検証ツールが必要なセクション構造を強制します。

コンテキスト効果がないパッケージ、またはコンシューマーが所有するパスが1つあるパッケージは、[`SENTENCE_MODEL_EXPERIENCE`](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/verify-package-readme-model-experience.ts) の監査済み `None, as ` または `Indirectly, through ` 文を使い、その後に `KV Cache effect` H4 と空でない1つの段落を続けます。モデル非依存の汎用パッケージは、代わりに `NO_MODEL_EXPERIENCE_SECTION` に加わってもかまいません。どちらの場合も、別のパッケージの作業の説明に拡張してはいけません。制限の[許可リスト](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/verify-package-readme-limitations.ts)は独立しています。アーカイブされた [Model Experience Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/archived/process/2026-07-12-package-model-experience-contract.md) に根拠が記録されています。

<a id="plugin-display-metadata"></a>

## 5. 任意のプラグイン表示メタデータを追加する

npm パッケージのプラグインでは、タイトルと説明を `locale/en.json` で定義します。`locale/zh.json` のような他の言語ファイルも同じフィールドを使います:

```json
{
  "meta": {
    "title": "Workspace Tools",
    "description": "Tools for your workspace."
  }
}
```

これらのエントリーを `package.json` にマージし、既存の実行時 export と公開ファイルを保ちます:

```json
{
  "exports": {
    "./package.json": "./package.json",
    "./locale/*.json": "./locale/*.json"
  },
  "files": ["locale/*.json"]
}
```

言語ファイルは `locale/` にまとめ、`en.json` を発見用のエントリーにします。フィールドは任意です。存在する値は空でない文字列でなければなりません。英語の `meta.title` と `meta.description` は sentence case を使います。最初の単語、固有名詞、製品名だけを大文字にします（例: `Git worktrees`）（[ルール](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/dsh-prose-standard/SKILL.md#required-coverage-by-prose-location)）。実行時、ファイルやフィールドの欠落はフォールバックを許しますが、不正な JSON や無効なフィールドはプラグイン単位の診断を生成します。ファーストパーティ・パッケージとプラグイン・テンプレートは、`en.json` または任意の `zh.json` に存在するすべてのフィールドをカバーする `ja.json`、`es.json`、`fr.json`、`de.json`、`ru.json`、`pt.json` を提供しなければならず、プレースホルダー、スラッシュ・コマンド、API フィールド名を保持します。detail パスは既存の英語ドキュメントを指してもかまいません。`locale/<subpath>/` 以下のサブパス・リソースも同じチェックに従います。英語の説明はユーザーに見える振る舞いを説明し、文末の句読点で終わります。`pnpm run check:i18n` を実行して JSON メタデータと UI 辞書の完全性を検証します。JSON チェックは `check:all`、CI の静的チェック、hygiene でも実行されます。

フィールドは、ビュー固有の名前整形の前に、既存の locale 言語チェーンを最初に使って独立にフォールバックします:

- タイトル: locale の `meta.title` → パッケージ・ルート・プラグインなら `package.json.name` → 完全な Cordis プラグイン名。
- 説明: locale の `meta.description` → パッケージ・ルート・プラグインなら `package.json.description` → 説明なし。

locale の lookup 用に `<package name>/locale/en.json` を export し、パッケージ・フィールドのフォールバックやアイコン宣言用に `<package name>/package.json` を公開します。`<package name>/search` のようなサブパス・プラグインはパッケージではなく、`./search/package.json` で export されたものでも `package.json` を決して読みません。テキストには `./search/locale/*.json` を、画像には `./search/icon` を使います。

バンドル・カード、詳細、コンポーネント行の画像では、パッケージ・ルート・プラグインは export するマニフェストのトップレベルに `"icon": "./icon.svg"` を設定するか、そのフィールドが省略された場合に `./icon` を export します。サブパス・プラグインは独自のアイコンを export します（例: `"./search/icon": "./search.svg"`）。各画像を `files` に含めます。マニフェストの `icon` はマニフェストのディレクトリからの相対パスです。絶対パス、URL、そのディレクトリ外のパス、その外に解決される symlink は拒否されます。export されたアイコンは locale リソースと同様に Node の exports を通じて解決され、パッケージ内に留まらなければなりません。SVG、PNG、JPEG（`.jpg`/`.jpeg`）、WebP が 256 KiB までサポートされます。画像は自己完結していなければなりません。SVG はインライン HTML ではなく画像としてレンダリングされます。Host はプラグインを有効化せずに data URL を返します。無効な宣言や読めないファイルは、export を試みずにメタデータ診断を生成し、有効なテキストは保持します。画像が欠落しているかデコードできない場合は、パネルの既定のアートワークを使います。

バンドルは `meta.detail` をパッケージ・ルートからの相対 Markdown ファイル・パスに設定することもできます。たとえば `locale/en.json` の `"docs/DETAIL.md"`、`locale/zh.json` の `"docs/DETAIL.zh.md"` です。これはパッケージ・ルートでこれらのファイルから読み取られます。パスは `/` を使い、パッケージ内に留まり、`.md` または `.markdown` で終わり、パッケージ・フィールドのフォールバックはありません。画像はその隣に置き、`check-workspace-constraints` が要求するように、その最上位ディレクトリ（例: `docs`）を locale リソースの直後の `files` に列挙します。指定されたファイルはパッケージのサブディレクトリに存在しなければなりません。ルート直下のドキュメントは、隣接する画像が公開されないため拒否されます。**Plugins** 内のバンドルのページ、および[カスタム・プラグイン索引バンドル](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/plugin-market-bundle/README.md)が有効な場合のその索引エントリー・ページは、アクティブな UI 言語のファイルをプラグインの詳細ドキュメントとして表示します。`meta.detail` がない場合、それらはパッケージ README を表示し、索引エントリー・ページは宣言されたリポジトリの README にフォールバックします。[ドキュメント順序](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/util/plugin-detail/README.md#document-order)がルールを定義します。ユーザー向けに書き、画像はファイルからの相対パスで参照します。

インストール済みバンドルのカードと詳細、コンポーネント・リストと設定詳細、Settings のプラグイン・インベントリーは、無効化されたプラグインやプリセット・プラグインも含めてこのメタデータを表示します。読み取りはプラグインを有効化しません。

リテラルのパッケージ名とモジュール名のフォールバックを短縮するのは Settings だけです。npm スコープと Cordis/DSH プレフィックスを削除します。Plugin Manager は完全な名前を保ちます。locale のタイトルと説明は変更されません。行の設定ページは、プラグインに表示用の説明がない場合、登録された summary を使えます。

Install ビューは引き続き `pnpm view` からの npm レジストリ情報を使い、locale メタデータは使いません。

結果を検証します:

1. リポジトリ・ルートから `pnpm run verify-package-meta` を実行し、フィールド、リソースの export、公開範囲を確認します。
2. インストール済みプラグインの該当する Plugin Manager と Settings のエントリーを英語と中国語で切り替え、タイトル、説明、フィールド単位のフォールバック、Settings のみの短縮された技術名を確認します。

## 6. 検証する

```sh
pnpm install        # registers the workspace
pnpm run doc-sync
pnpm run constraints && pnpm run typecheck && pnpm run lint
pnpm run build && pnpm run hygiene
```

新しいパッケージに必要な振る舞い固有のチェックとカバレッジについては、[リポジトリのテスト方針](../testing.md)に従ってください。
