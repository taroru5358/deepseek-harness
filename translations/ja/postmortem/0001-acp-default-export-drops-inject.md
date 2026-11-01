# ポストモーテム 0001: 接続時に ACP サーバーがクラッシュ — `export default` がプラグインの `inject` を落とした

日本語 | [English](../../../docs/postmortem/0001-acp-default-export-drops-inject.md) | [中文](../../../docs/postmortem/0001-acp-default-export-drops-inject.zh.md)

> これは非公式の日本語訳です。英語版（[docs/postmortem/0001-acp-default-export-drops-inject.md](../../../docs/postmortem/0001-acp-default-export-drops-inject.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

ステータス: 解決済み（PR #41 `feat/acp-2-bridge` で修正）

## エグゼクティブ・サマリー

完全なユニットテストカバレッジがあったにもかかわらず、2つの統合上の誤りが ACP を壊した。default export により Loader が `inject` を破棄し、トレースされたオプショナルサービスの参照が shadow 境界を越えて失敗した。手動マウントのテストはどちらの経路も通っていなかった。修正では、キー不要の実 Loader カバレッジと、プラグインの export およびオプショナルサービスアクセスに関するパッケージルールを追加した。

## 概要

ACP サーバー（`dsh --profile acp`、`@deepseek-ai/dsh-acp`）は、実際のエディタ（Zed）が接続した瞬間にクラッシュした。最初の `session/new` リクエストが `Internal error: cannot get property "agents" without inject` を返し、`session/load` も `sessionPersistence` について同じエラーを返した。178 件のユニットテストが通っていて行カバレッジも 100% だったにもかかわらず、ブリッジは本番で完全に機能しなかった。同じエラー文字列の裏に2つの独立したバグが隠れており、テストスイートは同じ理由で両方を見逃していた。すべてのテストが、プラグインが実際にどうロードされ、サービスが実際にどう解決されるかをたどらない経路でプラグインをマウントしていたのだ。

## 影響

ACP サーバーは1つのセッションも作成・ロードできなかった。エディタが最初に呼ぶ2つの RPC である。agent を Zed に接続しようとした人は誰でも即座に致命的な失敗を得た。データ損失はない（クラッシュ前に何も永続化されていない）。コストは完全に「機能が動かない」ことと、その理由を突き止めるデバッグ時間、しかも2回分だった。

## タイムライン

- ブリッジ（RFC 010）は、codec、インメモリ転送、生成されたプロトコルメッセージ、失敗経路、HMR の完全なユニットスイート、キーでゲートされた実 API の e2e、キー不要の stdout 純度 e2e とともにマージされた。すべて成功、カバレッジ 100%。
- 実際の Zed セッションは `session/new` で直ちに `cannot get property "agents" without inject` で失敗した。
- 調査は当初 Cordis の「traceable/shadow」説を追った（もっともらしく、メカニズム自体は実在する。Bug #2 を参照）。その後、vendored な `reflect.ts` の実際の fiber 走査に計装を入れ、実際のサブプロセスを実行した。トレースは、`apply()` の179行目で *プラグインのロード時* に、shadow のない ROOT fiber 上で throw していることを示し、`session/new` については shadow 説を反証した。
- 根本原因 #1 を発見: 紛れ込んだ `export default apply`。これを削除すると `session/new` は修正された。
- 削除すると Bug #2 が露呈した。`session/load` は依然として `sessionPersistence` で throw していた。これは本当に別のメカニズム（shadow 走査）であり、修正を分離して実際のサブプロセスを再実行することで確認した。

## 根本原因 #1 — `export default apply` がプラグインの `inject` を落とす（`session/new` が壊れた）

`packages/acp/acp/src/index.ts` は *namespace plugin* である。リポジトリ内の他のすべてのプラグイン（`invariants`、`llm-deepseek`、`tool-bash`、`tui`、…）と同様に、`name`、`inject`、`Config`、`apply` を個別の named export として公開している。しかし *さらに*、他のどのプラグインにもない1行が末尾にあった。

```ts ignore-check
export const name = 'acp'
export const inject = ['agents', 'sessions', 'sessionPersistence']
export function apply(ctx: Context, config: AcpConfig): void { /* … */ }
// …
export default apply   // ← the bug
```

プラグインが `cordis.yml` からロードされるとき、cordis Loader はインポートされたモジュールを `Loader.unwrapExports`（`vendor/loader/src/index.ts`）で正規化する。

```ts ignore-check
unwrapExports(exports: any) {
  if (isNullable(exports)) return exports
  exports = exports.default ?? exports        // ← prefers `.default`
  if (!exports.__esModule) return exports
  return exports.default ?? exports
}
```

default export が存在すると、`exports.default ?? exports` は **裸の `apply` 関数** に解決される。裸の関数には `inject`、`name`、`Config` プロパティがない。それらはモジュール名前空間上の *兄弟* の named export として存在しており、`.default` への展開が名前空間を捨ててしまった。Loader は空の `inject` からプラグインの fiber を構築した。

その結果、`apply` は **注入されたサービスがない** fiber で実行された。最初の行 `const agents = ctx.agents` が fiber ツリー（ROOT → Include → Loader → ROOT）を走査し、どの fiber の store にも `agents` を見つけられず、ルート fiber（`runtime === null`）に到達して `cannot get property "agents" without inject` を throw した。クラッシュは *ロード時* に起きており、後続のリクエストハンドラではない。失敗したトレースでは、たまたまそのリクエストがロードのトリガーだっただけだ。

**修正:** `export default apply` を削除する。Loader はモジュール名前空間を使い、`inject`/`name`/`Config` を尊重し、`apply` は宣言されたサービスを実際に付与する fiber の内側で実行される。

## 根本原因 #2 — オプショナルサービスの読み取りが traceable shadow 経由で inject ガードに掛かる（`session/load` が壊れた）

#1 を修正すると `session/new` は動いたが、`session/load` は依然として `cannot get property "sessionPersistence" without inject` を throw した。こちらは *まさに* Cordis の traceable/shadow メカニズムであり、正確に理解する価値がある。

`session/load` は `agents.resume(...)` を呼び、それが `AgentLoop.resume()` に委譲され、そこで `this.ctx.sessionPersistence` を読んでいた。`AgentLoop` の `static inject` は意図的に `sessionPersistence` を含んでいない。注入すると、永続化しないデモが、決してロードされないバックエンドを待って永久に pending になるからだ。このサービスは別の兄弟プラグイン／fiber が提供しており、機会的に読まれる。

Cordis のサービスアクセスはコンテキストプロキシ（`vendor/cordis/src/reflect.ts`）を経由する。外部の fiber から取得した *traceable proxy* を通じてサービスのメソッドが呼ばれると（ここでは、ブリッジ fiber が `ctx.agents.resume` を呼び、レジストリが `this.factory`、すなわち `AgentLoop` を、呼び出し元に束縛した新しい traceable proxy として包み直して返す）、`createShadowMethod`（`vendor/cordis/src/utils.ts`）が `this` を *shadow* オブジェクトに再束縛する。その `ctx` は `[symbols.shadow]` を持ち、`AgentLoop` 自身の構築コンテキストを指す。すると `resume` の内部では、`this.ctx.sessionPersistence` の解決時にプロキシハンドラが shadow の fiber から fiber 走査を開始する。

```ts ignore-check
// reflect.ts get handler
let fiber = (ctx[symbols.shadow] as Context ?? ctx).fiber   // ← starts at AgentLoop's fiber
while (true) {
  const impl = fiber.store?.[prop]
  if (impl) return getTraceable(ctx, impl.value)
  if (prop in fiber.inject) { /* inactive-context error */ }
  if (!fiber.runtime) throw error                            // ← reached root, throw
  if (fiber.parent[symbols.isolate][prop] !== key) throw error
  fiber = fiber.parent.fiber                                 // ← ancestor-only
}
```

走査は **祖先のみ** である。`sessionPersistence` は `AgentLoop` の fiber store にも（`static inject` にないため）、ルートへ向かうどの祖先にも存在しない（*兄弟* ブランチに存在する）。したがって走査はルート fiber に到達して throw する。

なぜインメモリの `AgentLoop` resume テストはこれを捉えなかったのか。テストコードから *どのプラグイン fiber の外側* で `ctx.agents.resume(...)` を直接呼んでいたからだ。そこでは `ctx.fiber.runtime` が `null` なので、プロキシハンドラは早期バイパスを通る。

```ts ignore-check
if (!ctx.fiber.runtime) return ctx.reflect.get(prop, false)   // ← direct global-store lookup, no fiber walk
```

`ctx.reflect.get(name, false)` は isolate シンボルをキーにしたグローバルサービスストアへの直接参照であり、fiber のトポロジーを完全に無視してサービスを見つける。したがってトップレベルのテストからは読み取りが成功し、shadow 経由で到達した実際のプラグイン fiber の内側からは throw する。ブリッジはまさに後者である。

**修正:** オプショナルサービスは `ctx.get('sessionPersistence')` で読む。これはアクティブ状態のチェックを保ちつつ、isolate キーのグローバルストアを使う。プラグインが宣言した注入集合に含まれるサービスについては、直接のプロパティ参照が引き続き適切である。

## なぜすべてのテストが見逃したか（本当の失敗）

両方のバグは1つのプロセス上の欠陥を共有している。**どのテストも、プラグインを実際のロード経路や実際の呼び出しトポロジーで動かしていなかった。**

- インメモリ harness はプラグインオブジェクトを手で組み立ててブリッジをマウントする。`ctx.plugin({ name, inject, apply })` だ。これでは `inject` を手動で与えるため、Bug #1 を再現できない。`unwrapExports` は *Loader* だけが呼び、`ctx.plugin` は決して呼ばない。`ctx.plugin(NamespaceImport)` でも捉えられなかっただろう。
- 同じ harness はすべてを1つのルートコンテキストにフラットにマウントするため、そこから到達した `AgentLoop` の resume は、トップレベル（`!runtime` バイパス）で実行されるか、起点が依然としてルートで解決される shadow を経由する。これが Bug #2 の祖先走査の失敗を覆い隠した。
- 唯一のキー不要 e2e は `initialize` を送り、stdout の純度を確認していた。`initialize` はファクトリに到達しないため、両方のバグを素通りした。
- `session/new`/`session/load` を動かす唯一のテストはキーでゲートされていたため、CI（キーなし）ではスキップされた。ローカルでは、古いコードを含む古いビルド済み `lib/` がたまたまモジュール解決を満たしたから「通って」いただけだ。

行カバレッジ 100% はずっと満たされていた。カバレッジが証明するのは行が *実行された* ことであり、機能が *出荷される形で* 動くかどうかは何も語らない。

## 追加したガードレール

- **`export default apply` を削除**（`packages/acp/acp/src/index.ts`）— Bug #1 の修正。
- **`AgentLoop.resume` が `this.ctx.get('sessionPersistence')` を読む**（`packages/core/agent-loop/src/index.ts`）— Bug #2 の修正。shadow 走査の罠を説明するコメント付き。
- **実際の stdio を使ったキー不要の `session/new` e2e**（`apps/cli/tests/profiles/acp/tests/acp.e2e.ts`）: 実 Loader 経由でプロファイルをサブプロセスとして起動し、`session/new` が解決することを assert する。API キーなしで Bug #1 に対して明確に失敗する。`export default apply` を戻すと失敗することも確認済み。
- **e2e の spawn における `TSX_TSCONFIG_PATH`**: サブプロセスは一時的な cwd から実行され、tsx は上方探索でリポジトリルートの tsconfig `paths` マップを見つけられない。そのため dsh-* のインポートは暗黙にビルド済み `lib/` へフォールバックしていた。tsx にリポジトリの tsconfig を指すことで解決が cwd 非依存になり、テストが（古い可能性のあるビルドではなく）*ソース* を実行することが保証される。
- **[docs/testing.md](../testing.md) のルール**: 「実際のエントリパスをテストする」。行カバレッジは振る舞いのカバレッジではない。この教訓を今後のすべてのプラグインのために成文化する。

## 教訓

- cordis Loader の下では、namespace plugin と default export は排他である。namespace 形式（`name`/`inject`/`Config`/`apply`）を選び、`export default` を追加しないこと。`unwrapExports` が名前空間を捨ててしまう。
- プラグインが機会的に読み取り、`static inject` で宣言しないサービスには `ctx.get(name)` を使い、`ctx.<name>` は決して使わない。プロパティプロキシは祖先のみの fiber 走査で解決するため、外部の shadow を経由すると失敗する。`ctx.get(name)` はトポロジー非依存の参照である（かつデフォルトで strict。非アクティブなバックエンドは、teardown の途中で返されるのではなく `undefined` として読まれる）。
- プラグインを手で構築するテストは、プラグインがどうロードされるかを検証できない。少なくとも1つのテストが実際の Loader/export 経路をエンドツーエンドで動かす必要がある。主要な操作がモデルを呼ばないなら、そのテストに API キーは不要である。したがってキーゲートの背後ではなく CI に置くべきだ。
- 理論ではなくトレースを信じる。優雅な shadow の説明は本物だったが、それは *2つ目* のバグだった。*1つ目* は1行の export の誤りであり、もっともらしいが誤った推論に数時間を費やした後、fiber 走査の `console.error` が数分で見つけた。
