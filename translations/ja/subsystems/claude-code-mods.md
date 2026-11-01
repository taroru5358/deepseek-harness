# Claude Code mods: 互換性

日本語 | [English](../../../docs/subsystems/claude-code-mods.md) | [中文](../../../docs/subsystems/claude-code-mods.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/claude-code-mods.md](../../../docs/subsystems/claude-code-mods.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

このブリッジ経由で mod が Claude Code 上と異なる動作をする箇所を、Claude Code 2.1.287（[mods リファレンス](https://code.claude.com/docs/en/plugins/mods/reference)とその `claude-code.d.ts`）および 2026-10-01 の [Getting started with Claude Code mods](https://claude.dev/blog/getting-started-with-claude-code-mods/) 記事の時点で網羅する。各行は Claude Code の動作、このブリッジの動作、そして理由またはフォローアップを示す。ここに載っていないものは参照どおりに動作する。このホストが決して発火しないイベントへのフックは、mod のロード時に警告で示される。また、提供テーブルにない `$` 呼び出しは `no implementation for <namespace>.<method>` で拒否される。

## パッケージングとロード

| Claude Code | このブリッジ | 理由またはフォローアップ |
|---|---|---|
| `.claude-plugin/plugin.json` と、1 つの hooks モジュールを指定する `hooks/hooks.json` を持つプラグイン・ディレクトリ；`claude --plugin-dir`、マーケットプレイス、`/plugin install` | mod はプラグインである。`defineMod({ name, version, root, userConfig, register })` が hooks モジュールをラップし、ブリッジの後に cordis.yml 経由でマウントする。`plugin.json` と `hooks.json` は読まれない | mod はすべての DSH プラグインが使う単一のコンポジション機構に参加する。サンプル・ディレクトリはマニフェストを保持するので、`claude --plugin-dir` でも動作する |
| `plugin.json` 内の型付きデフォルトを持つ `userConfig` スキーマ；ロード時にオプションを検証 | `userConfig` は `defineMod` スペックのデフォルト・オブジェクトである。プラグインの cordis.yml `config` がそれを上書きし、文字列・数値・真偽値・文字列リストのレコードとして検証される | マニフェストは読まれない |
| 階層（`user`、`project`、`managed`）と `next.to(e, tier)` | すべての mod は `user` としてロードされる。テスト内の `tier()` は受け入れられて無視され、`next.to` は拒否される | managed 設定は DSH の概念ではない |
| 編集時の hooks モジュールのホットリロード；`claude plugin validate`、`claude plugin test`、`types/index.d.ts` からの型生成 | これらのコマンドはない。再マウントは同じ評価済みモジュール上で `register` を再実行するので、モジュール・レベルの変数は値を保つ | フォローアップは要求時のみ。テストキットの `createModTestKit` が Vitest 内で `claude plugin test` を置き換える |
| hooks モジュールは TypeScript でもよい | `.ts` の hooks モジュールは、ランチャーがトランスパイルする場所でのみロードされる（ソース起動ではロードされ、ビルド済みインストールではロードされない） | DSH は素の Node を同梱する |
| mod はプロセス内で動作し、`$` だけがホストへのアクセス手段である | hooks モジュールはプロセス内で Node のグローバルとともに動作し、アクセス規則はなく、プロセスの完全な権限を持つ。`$.env` は harness 環境を読み書きし、`$.http.fetch` は任意の URL に到達し、`$.fs` と `$.tool.call` はセッションとして動作する | mod にはサンドボックスが適用されない。プラグインとして実行してよい mod だけをマウントすること |
| `import type { … } from 'claude-code'`、`declare module 'claude-code' { interface PluginState }` | `claude-code` モジュール名はこのリポジトリのテスト・セットアップ内でのみ、ブリッジの型に解決される。その外の mod は `@deepseek-ai/dsh-experimental-claude-code-mods` をインポートする | 型名はブリッジ自身のものである |
| `hooks.json` の `settings hooks` が mod のフックと並んで実行される | 実行されない。それらには `@deepseek-ai/dsh-hooks-claude-code` をマウントする | 別のブリッジである |

## イベント

| イベント | Claude Code | このブリッジ |
|---|---|---|
| `session.start` | セッションごとに 1 回、最初のプロンプトの前に発火する | ルート agent に対して `agent/created` から発火し、最初のターンの前に待機される。サブ agent では発火しない |
| `session.end` | セッションごとに 1 回発火する | ルート agent に対して `agent/disposed` から発火する。`$.state` はフックが落ち着くまで読める |
| `prompt.submit` | `e.text` は入力されたプロンプトである。`context` の各エントリーは、入力されたプロンプトの後にそれぞれ 1 ブロックとなる。プラグインが送信したプロンプトは `origin: { kind: 'plugin', name }` を持つ | 同じ。`e.text` はクレームされたバッチ内で人間自身の（`user` 由来の）メッセージのテキスト・ブロックを結合する。注入されたコンテキストだけのバッチでは何も発火しない。`$.prompt.submit` がキューに入れたプロンプトは、送信した mod を origin として持つ |
| `turn.start`、`turn.complete` | ターンごと。サブ agent のターンは `agentId` を持つ | 同じ。`turn.complete` の `{ text }` はホスト・ログに届き、回答の下の行にはならない。`durationMs` は `turn/start` から数える |
| `tool.call` | Claude Code の権限チェックの前に実行される。`next({ ...e, command })` はツールが実行する引数を書き換え、`next({ ...e, tool })` は呼び出しの経路を変える | `tools/execute` の周りで、harness の権限判断の後に実行される。引数を書き換えたり別のツールを指名したりするフック（または `.catch` ハンドラー）はレポート付きでスキップされる。引数の場合は[実行前入力書き換えの提案](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/proposed/feature/2026-06-30-pre-tool-input-rewrite.md)を名指しする。ログに記録された呼び出しは記録どおりに実行されるためである。`{ deny }`、`{ result }`、観察、および `next` 後の結果や `isError` の書き換えは記述どおりに動作する |
| `command.run` | mod が登録したコマンドに対して実行される | 同じ |
| `ui.render` | すべてのレンダー・サイト（`AbovePrompt`、`Pane`、`ToolUse`、`AssistantMessage`、`Spinner`、…）に対して発火する | `AbovePrompt` に対してのみ、セッションのバンドの再描画ごとに 1 回発火する。`Pane` フックはテストキットの `$.ui.mount({ component: 'Pane' })` 経由でのみ実行される |
| `tool.check`、`tool.describe`、`tool.list`、`tool.register` | ツール・リストと権限を形成する | 発火しない。`$.tool.register` と `$.tool.list` は `$` 呼び出しとして提供される |
| `turn.step` | モデルのステップごと | 発火しない |
| その他すべての参照イベント：他の `prompt.*` イベント、`skill.prompt`、`attribution.text`、`command.describe`、`config.*`、他の `session.*` イベント、`agent.*`、`plugin.register`、`engine.create`、`telemetry.*`、`ui.resolve`、`ui.press`、`ui.input`、`ui.select`、`ui.focus`、`ui.scroll`、`ui.close`、`ui.message` | エンジンによって発火される | 登録されるが決して発火されず、ロード時の警告で示される |
| `classic.*` | イベントとしての classic 設定フック | 未知のイベント名と同様、`register` で拒否される |
| `<namespace>.<method>` の op イベント（`fs.read`、`store.set`、…） | 後にロードされた mod の `$` 呼び出しは、それより前にロードされた mod に到達する | 同じ。`tool.call` は代わりにツール・パイプライン経由で届く |

## チェーン

| Claude Code | このブリッジ | 理由 |
|---|---|---|
| フックは `next` を複数回呼べる。各呼び出しは下のチェーンを再び実行する | `next` は 1 回だけ実行される。2 回目の呼び出しは最初の実行の結果を返す | 記録された呼び出しにつき 1 回のツール実行 |
| `next` がまだ実行中のときにフックが応答する | 下の実行はフックの応答が確定する前に待機される。応答後に下で失敗した場合は報告される | イベントは、それが開始した何も実行されていないときにのみ確定する |
| 自身の時間の予算は 10 秒、`.catch` は 1 秒。`$.clock.sleep` を除き、`next` と `$` 呼び出し内の時間は除外される | 同じで設定可能（`hookTimeoutMs`、`catchTimeoutMs`） | — |
| 結果オブジェクトを返さないことは失敗である | 同じ。`null` は応答として受け入れられる（空に描画されるサーフェス） | 描くものがない `ui.render` |
| 結果の未知のフィールドは破棄され、誤った型はフックを失敗させる | 誤った型を持つ `prompt.submit` の結果は、元のテキストと文字列のコンテキスト行だけを保つ | 部分的な結果が一般的な唯一のイベントでは寛容である |

## `$` メンバー

| 名前空間 | Claude Code | このブリッジ |
|---|---|---|
| `$.plugin` | `name`、`root`、`tier` | `defineMod` スペックの `name`、`root` |
| `$.ui` | `resolve`、`invalidate`、`open`、`close`、`panes`、`focus`、`log`、`toast`、`status`、`notice`、`ask`、`blit`、`copy`、`press`、`input`、`select` | `resolve`、`invalidate`、`open`、`close`、`panes`、`log`、`toast`、`status`、`ask`。`open` は理由付きで `{ isPlaced: false }` を返し、バンドを再描画する。`log`、`toast`、`status` はホスト・ログに届く。残りは拒否される |
| `$.command` | `register`、`run`、`list` | 同じ。誰も応答しないコマンドの `run` は、登録した mod を名指しして拒否される |
| `$.tool` | `register`、`call`、`list` | 同じ。登録されたツールは `mcp__<plugin>__<tool>` である。`call` は harness パイプラインを実行し、ツールの遅延コンテキストをセッションに注入する |
| `$.prompt` | `submit`、`compose` | `submit`。メッセージは `user` 由来で、`asUser` でない限り `Message from the "<plugin>" mod:` という枠が付く。それが発火する `prompt.submit` は mod を origin として名指しする |
| `$.session` | `id`、`cwd`、`root`、`repo`、`model`、`turns`、`messages`、`usage`、`version`、`send`、`append`、`authorize`、`compact`、`surfaces` | `id`、`cwd`、`root`、`model`、`turns`、`messages`、`usage`、`version`。`cwd` はコミット済みの現在ディレクトリを報告する。`root` は元のプロジェクトを保持する。`usage.context.window` は `0` で、トークン・メーターがルートのウィンドウを知るまで `percent` は存在しない。`rateLimits` は空である。`version` はこのブリッジを名指しする |
| `$.state` | セッションごとで、ホットリロードを越えて存続する | セッションごとにメモリ内に存在する。`ui.render` 中の読み取りは、バンドをそのスロットに購読させる |
| `$.store` | プラグインごとで、永続的 | 同じで、`claude_code_mods` ストレージ・ドメイン内、プラグインあたり 4 MiB |
| `$.clock` | `now`、`sleep`、`after`、`every`。タイマーはプラグインとともに終了する | 同じ。タイマーはまた、そのイベントをスケジュールしたセッションとともに終了し、イベントがキャンセルされると `sleep` は拒否する |
| `$.fs` | `read`、`write`、`list`、`exists`、`stat`、`ancestors` | `ancestors` 以外のすべて。ファイルあたり 4 MiB。`stat.mtimeMs` は `0` |
| `$.process` | `run`、`spawn` | `run` のみ。argv はシェルを介さず、デフォルトは `processTimeoutMs` |
| `$.http` | `fetch` | 同じ。ボディは最大 4 MiB |
| `$.env` | `get`、`set` | 同じ。プロセス環境はすべてのセッションとプラグインで共有される |
| `$.model`、`$.agent`、`$.config`、`$.settings`、`$.mcp`、`$.audio`、`$.telemetry`、`$.turn` | 提供される | `no implementation` で拒否される。`$.model.complete` は記録された side-request イベントを待つ |

## 描画

| Claude Code | このブリッジ |
|---|---|
| `Box`、`Text`、`Button`、`Input`、`Spacer`、およびホスト自身の要素種別 | `$.ui.resolve(e)` からの `Box`、`Text`、`Button`。ツリーは検証され、`Button` の `onPress` は描画ごとのアクション id の背後でホスト内に留まる |
| `Box` のプロパティ：`flexDirection`、`padding*`、`gap`、`border*`、`width`、`height`、`overflow`、`scroll` | `flexDirection`、`padding`、`paddingX`、`paddingY`、`gap`、`border`、`borderColor`。その他はシリアライズされ、Web バンドはそれらを無視する |
| `Text` のプロパティ：`color`、`bold`、`dimColor`、`italic`、`underline`、`wrap`、`truncate` | `color`（ターミナル・パレット名）、`bold`、`dimColor`、`italic`、`underline`。Web バンドは折り返す |
| `Button` のプロパティ：`label`、`hotkey`、`disabled`、`onPress`。素の数字がホットキーを武装する | `label`、`hotkey`（ヒントとして表示）、`disabled`、`onPress`（クリック） |
| バンドは 1 つのインスタンスである。`hasSurvey`、`isWorking`、`maxRows`、`bodyColumns` はターミナルを反映する | セッションごとに 1 つのインスタンスで、mod のロード順に参照される。`bodyColumns` と `viewport.columns` はブリッジの `bandColumns`（デフォルト `120`）、`maxRows` はその `bandRows`（デフォルト `10`）であり、`hasSurvey` と `isWorking` は `false` である。Web バンドはスクロールせずに折り返す |
| バンドはホストのフレーム時と、レンダーが読んだ `$.state` の値が変化したときに再描画される | `session.start` の後、最後の描画が読んだ `$.state` の値が書き込まれたとき、`$.ui.open`、`$.ui.close`、`$.ui.invalidate` の後、各 `tool.call` チェーンと `turn.complete` が落ち着いた後、そして押下の後に再描画される。他のタイミングでモジュール・レベルの状態を変える mod は、次のトリガーでそれを描画する。押下は時間制限なしで `onPress` を実行する |
| `focus`、`scroll` を持つ `Pane` がトランスクリプトの脇にドッキングされる | ペインはない。`$.ui.open` が `{ isPlaced: false }` を返すので、mod はバンドにフォールバックする。テストキットは `Pane` フックを直接マウントする |
| ホットキー、`TextInput`、フォーカス、スクロール、`ui.press`、`ui.input`、`ui.select` | 提供されない。フォローアップ |

## テスト

| `claude-code/testing` | このブリッジ |
|---|---|
| `describe`、`test(name, ($, on) => …)`、`expect`、`mock`、`tier` | このリポジトリ内では同じモジュール名が Vitest 上でこれらを提供する。`test` はテスト・ファイルの位置（`<mod>/tests/*.test.ts` → `<mod>/index.ts`）から mod を推論し、`defineModTests` がそれを上書きする。外部では `createModTestKit` と `mock` は `@deepseek-ai/dsh-experimental-claude-code-mods/testing` から来る |
| `on` はエンジンの応答をスタブし、キットは `ui.*` に自身で応答する | 同じ。`ui.open` は `{ isPlaced: false }` を返し、`ui.invalidate` と `ui.close` は成功する。`session.cwd`、`process.run`、`fs.*`、`store.*`、`env.*` にはスタブが必要である（`mock.store`、`mock.env`、`mock.clock` は名前空間全体に応答する） |
| `$.ui.mount({ plugin, surface, component, props })` → `find`、`findAll`、`press`、`unmount` | 同じで、加えて `tree()` と `text()`。すべての読み取りは、ロードされたすべての mod のフックを通じて新たにレンダーされる。`plugin` と `surface` はソース互換性のために受け入れられる |
| テスト内のフック予算は 5 秒 | 同じ（`budgetMs`） |

## サンプル mod

| mod | ソース | このブリッジ経由 |
|---|---|---|
| Token Weather | 記事のモジュール、型契約、テストを変更なしで使用 | 公開されたまま動作する。テストの 3 つの `as any` キャストは、キットが型付きであるため削除されている。トークン・メーターを通じて `$.session.usage` からバンドに描画する |
| Blast Radius | 記事の `tool.call` フックを変更なしで使用。`classify`、`measure`、ツリーは記事の説明に合わせて完成させた | 実際の `sleep` ポーリングでコマンドを保持し、レポートをバンドに描画し（`isPlaced: false`）、Proceed か Cancel で決定する。Token Weather より前にロードすること |
| Replay Theater | 記事の 5 つのフックを変更なしで使用。`stepsFor`、diff、`openReplay`、ペイン、ヒントは完成させた | Edit と Write の呼び出しを記録し、ターン後にバンドでヒントを出し、ペインが配置されないためバンドで diff をステップ実行する |
| `diff`（[anthropics/claude-code](https://github.com/anthropics/claude-code/tree/main/mods/diff)、`52c7644` で読んだもの） | TypeScript、約 570 ファイル、フックは `command.run`、`prompt.submit`、`session.start`、`tool.call`、`ui.close`、`ui.focus`、`ui.render`、`ui.scroll` | 実行不可。`$.settings.read`、`$.telemetry.*`、ペイン上の `$.session.messages`、`ui.focus` と `ui.scroll` イベント、配置された `Pane` を必要とする。読解により評価した。リポジトリの条項はフィクスチャのコピーを許さない |
| `agents-md`（同じリポジトリ） | フックは `agent.spawn`、`prompt.context`、`session.start`、`tool.call` | 実行不可。`prompt.context` と `agent.spawn` は発火されず、`$.fs.ancestors`、`$.telemetry.*` は提供されない |
| `sec-default`（同じリポジトリ） | フックは `tool.check`、`prompt.compose`、`settings.read`、`classic.*`、`plugin.register`、… | 実行不可。このホストが発火しないイベント上の managed 階層ポリシー |
| `telemetry`（同じリポジトリ） | フックは `engine.create`、`telemetry.*`、`session.start`、`session.end`。`$.session.repo`、`$.settings.read` を呼ぶ | 実行不可。`telemetry.*` と `engine.create` は発火されず、2 つの `$` メンバーは提供されない |

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxclaudecodemods--claudecodemods"></a>

### `ctx.claudeCodeMods` — `ClaudeCodeMods`

ブリッジ・サービス：ロード済みの mod、そのフック、およびそれらのイベントを発火する harness リスナー。mod は ClaudeCodeMods.add を通じて参加し、mod プラグインがマウントされるときに defineMod がこれを呼ぶ。Remote メソッドは、Client が各セッションのバンドをプロンプトの上に描画し、そのボタンを押せるようにする。

```ts cordis-catalog
/**
 * Watch the band above the prompt of one session: the current drawing, then
 * every redraw, until the Client stops watching.
 * @param agent - the session's agent, resolved by the Gateway.
 * @param signal - carrier cancellation.
 * @returns the band's snapshots.
 */
@Remote({ mode: 'stream' }) watchBand(agent: Agent, signal: AbortSignal): AsyncIterable<SurfaceSnapshot>

/**
 * Press a button of the band's current drawing: runs the mod's `onPress` and redraws.
 * @param agent - the session's agent, resolved by the Gateway.
 * @param generation - the drawing the Client saw.
 * @param actionId - the button's action id in that drawing.
 * @returns the snapshot after the press.
 */
@Remote pressBand(agent: Agent, generation: number, actionId: string): Promise<SurfaceSnapshot>

/**
 * Load one mod beneath every mod loaded before it: run its `register`,
 * keep its hooks, and report which of its events this host never raises.
 * @param definition - the mod as its plugin defined it.
 * @returns the disposer that removes the mod's hooks, closes its timers, and releases its registrations.
 * @throws Error when the name is invalid or taken, or when `register` throws (Claude Code's `hooks module did not load` wording).
 */
async add(definition: ModDefinition): Promise<() => Promise<void>>
```

型: [Agent](core.md)

ソース: [`packages/experimental/claude-code-mods/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/claude-code-mods/src/index.ts)
<!-- END GENERATED cordis-surface -->
