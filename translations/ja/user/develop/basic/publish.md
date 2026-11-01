# plugin をパッケージ化してインストールする

日本語 | [English](../../../../../docs/user/develop/basic/publish.md) | [中文](../../../../../docs/user/develop/basic/publish.zh.md)

> これは非公式の日本語訳です。英語版（[docs/user/develop/basic/publish.md](../../../../../docs/user/develop/basic/publish.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

前のチュートリアルでは、`--patch` オーバーレイ経由でローカル plugin を読み込みました。このチュートリアルでは、それをインストール可能な **bundle** としてパッケージ化し、`dsh plugin add` で **profile** にインストールし、合成後の設定を決めるレイヤー順序を説明します。`dsh` CLI がインストール済みであることを前提とします。先に [plugin の設定](config.md) を完了してください。

代わりに新しいソースチェックアウトを使う場合は、[ソースからの実行](../../../overview.md#run-from-source) を完了し、このチュートリアルの `hello-plugin` ディレクトリをリポジトリルートに置き、残りの `dsh ...` コマンドをそこで `pnpm dsh ...` として実行します。ビルドとランチャーの動作は [ソース実行](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/cli/reference/README.md#source-execution) を参照してください。

## 2つの概念、2つのマニフェスト

インストールは2つの概念に基づきます。どちらも `package.json` で記述されますが、`dsh` キーの下に異なる種類のマニフェストを持ち、異なる問いに答えます:

- **bundle** は設定レイヤーを同梱する npm パッケージです。そのマニフェストは `dsh.bundle` を宣言し、「このパッケージは何を提供するか」に答えます。plugin の行を挿入または上書きする patch ファイルです。
- **profile** は `$DSH_HOME/profiles/<name>` の下にあるディレクトリで、1つの実行可能な構成を表します。そのマニフェストは `dsh.profile` を宣言し、「どの bundle が、どの順序でこのセットアップを構成するか」に答えます。

bundle はあなたが作成・配布するもので、profile はユーザーが `dsh --profile <name>` で起動するものです。両方を兼ねるものはありません。

### bundle マニフェスト

パッケージディレクトリを作成します:

```sh
mkdir -p hello-plugin
```

```
hello-plugin/
├── package.json       # declares dsh.bundle
├── cordis.patch.yml   # the layer applied when a profile lists this bundle
└── index.js           # plugin modules the patch rows reference
```

`hello-plugin/package.json` を作成します:

```json
{
  "name": "dsh-hello-plugin",
  "version": "0.1.0",
  "type": "module",
  "main": "index.js",
  "files": ["index.js", "cordis.patch.yml"],
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } }
}
```

plugin のエントリポイントとして `hello-plugin/index.js` を作成します:

```js
export const name = 'hello-plugin'

export function apply() {
  console.log('[hello-plugin] plugin loaded!')
}
```

`hello-plugin/cordis.patch.yml` を作成します。patch は、これまでに書いた `--patch` オーバーレイと同じ YAML 配列です。ただし、plugin の行は相対ソースパスではなく名前でパッケージを参照するため、Node の解決がインストール済みコードを見つけられます:

```yaml
- insert:
    - id: hello
      name: dsh-hello-plugin
```

`patch` はファイルの順序付きリストも受け付けます。たとえば `["./base.patch.yml", "./web.patch.yml"]` です。ランチャーはそれらをその順序で1つのレイヤーとして適用し、各ファイルの相対 plugin パスはそのファイルの隣で解決されます。`dsh.bundle` 宣言のないパッケージもインストールできますが、単なる依存関係としてのみです。`dsh plugin` は警告を表示し、レイヤーを有効化しません。このパッケージ形式は、ユーザーが有効化する plugin ではなく、plugin パッケージがインポートするライブラリに使います。

`engines.dsh` の SemVer 範囲で、bundle がサポートする DSH リリースを宣言します。たとえば `"engines": { "dsh": ">=0.2.0 <0.3.0" }` です。インストールと profile の起動は、実行中の DSH が範囲を満たさない bundle を拒否します。`engines.dsh` がない場合は、代わりに `@deepseek-ai/dsh*` の peer 依存関係の範囲を確認します（[ルール](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/app-boot/README.md#profiles)）。

### profile マニフェスト

profile ディレクトリには2つのファイルがあります:

- `package.json` — profile の out-of-tree plugin 依存関係（pnpm が管理）と、順序付きの `bundles` リストを持つ `dsh.profile` マニフェスト。
- `cordis.patch.yml` — ユーザー自身の patch レイヤー。すべての bundle レイヤーの後に適用されます。

profile マニフェストを手書きすることはありません。`dsh --profile <name> --from-default-profile <template>` は同梱のアプリケーションテンプレートから作成またはリセットでき、`dsh plugin` は base を基盤とする profile を作成してインストール済み bundle リストを維持します。作成規則は [CLI 動作リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/cli/reference/README.md#profile-boot) が管理し、次のセクションでは plugin の経路を示します。

## profile にインストールする

`dsh plugin --profile <name> <args...>` は profile ディレクトリ内の pnpm に転送されるため、すべての pnpm 動詞が動作します。`hello-plugin` を含むディレクトリから、パッケージのチェックアウトをインストールします:

```sh
dsh plugin --profile demo add ./hello-plugin
```

初回使用時に profile が初期化され（最初の bundle として `@deepseek-ai/dsh-base`）、pnpm がチェックアウトをリンクし、パッケージが `dsh.bundle` を宣言しているため `dsh` が bundle を `dsh.profile.bundles` に追加します:

```json
{
  "name": "dsh-profile-demo",
  "private": true,
  "dependencies": {
    "dsh-hello-plugin": "link:/path/to/hello-plugin"
  },
  "dsh": {
    "profile": {
      "bundles": [
        "@deepseek-ai/dsh-base",
        "dsh-hello-plugin"
      ]
    }
  }
}
```

リンクされたチェックアウトは自身の `node_modules` を保持します。plugin がホストとインスタンスを共有する必要がある dsh パッケージは、harness パッケージと同様に `peerDependencies` と `devDependencies` の両方で宣言します。そのマニフェストの検索位置では、実行中の dsh の runtime 解決に存在する peer はインストール側のコピーを使います。devDependency のコピーは型チェッカーと単体テストに使われます。独立してバージョン管理されるサードパーティ依存関係と、状態を持たない dsh ユーティリティは `dependencies` に置きます。

通常のリンクされたインポートは Node の祖先順序に従い、各ディレクトリの現在の peer 宣言を確認します。より近い物理パッケージが、より上位の peer 宣言より先に優先されます。リンク先に `package.json` がなくても、そのマニフェストの隣に物理的な `node_modules` がなくても、祖先の peer は適用されます。明示的な `require.resolve(..., { paths })` は常にネイティブで、profile 内のパスも含みます。これらの規則は npm、Desktop、ソース起動で共有されます。読み込み済みモジュールを無効化したり、peer のバージョン範囲を検証したりはしません。スコープとファイルクエリの動作は [解決規則](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-19-profile-resolution-lookup-order.md) を参照してください。

広いチェックアウトをリンクしても、実行中のインストール自身のパッケージディレクトリに peer インターセプトは適用されません。pnpm ストアを含め、リンク先が profile 内にとどまるリンクは、外部リンクルートではなく profile 所有のインストールコンテンツのままです。重複する外部リンクは検索順序を変えません。各リクエストは依然としてそのインポーターのディレクトリから始まります。

起動せずにレイヤーを検証してから、起動します:

```sh
dsh --profile demo --dump-config   # shows a "# == dsh-hello-plugin" layer
dsh --profile demo
```

`dsh plugin --profile demo remove dsh-hello-plugin` は依存関係とレイヤーの両方を削除します。

## 読み込み順序

実効設定は、空のルートに対して次の順序で適用して合成されます:

1. profile の `dsh.profile.bundles` リストにある各 bundle patch。リスト順で、`@deepseek-ai/dsh-base` が最初、次にインストールされた各 bundle が追加された順です。
2. profile 自身の `cordis.patch.yml`。
3. ホームレベルの `$DSH_HOME/cordis.patch.yml` — すべての profile で共有されるマシンローカルの設定。
4. 各 `--patch <path>` オーバーレイ。argv の順序です。

アプリの引数は別の patch レイヤーではありません。surface bundle は、後述する通常のアプリ所有サービスを通じてそれらを解決できます。

後のレイヤーが行ごとに優先され、patch はキーを深くマージするのではなく、行の `config` 値全体を置き換えます。bundle の作者にとっての2つの結果:

- あなたの patch は、[`dsh-web-app` bundle](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/web-app/cordis.patch.yml) が `dsh-base` の行を上書きするのと同じ方法で、`id` によって前のレイヤーの行を上書きできます。ただし、変更したキーだけでなく、その行が必要とするすべてのキーを再指定する必要があります。
- ユーザーは自身の profile の `cordis.patch.yml` で、あなたのパッケージに触れずにあなたの行を上書きできます。そのため、ユーザーが維持しそうな設定既定値を優先し、残りはスキーマに任せてください。

同梱 bundle の名前は常に dsh インストール自体から解決されます。pnpm が管理するのは out-of-tree パッケージのみなので、あなたの bundle は `@deepseek-ai/dsh-base` が存在して最新であることを前提にできます。

## surface bundle に独自のコマンドラインを与える

実行可能なアプリを定義する bundle は、通常の provider plugin をマウントします:

```yaml
- id: hello-startup
  name: 'dsh-hello-plugin/startup'
```

plugin は `inject = ['cmdlineArgs']` をエクスポートし、自身の commander プログラムで [`@deepseek-ai/dsh-cmdline`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/cmdline/README.md) の `parseCmdline` を呼び、プログラムの action からアプリ所有サービスを提供します。ランチャーはランチャーフラグの後にある同じ不変の引数をすべての plugin に渡すため、アプリ固有のフラグにランチャーの変更は不要で、複数の plugin がそのスナップショットを解析できます。Loader の行にランチャーのマーカーや特別な種類は不要です。

これらの引数で設定される行は、provider のサービスを inject し、自身の `!!js` オプションから読み取ります。その隣のデプロイ値がフォールバックになります:

```yaml
- id: my-app
  name: '@example/my-app'
  inject: [myAppStartup]
  config:
    port: !!js ctx.myAppStartup.port ?? 8080
```

`--help` では provider がサービスを公開しないため、それらの行は有効化されません。Loader は構成を一度マウントし、各行の通常の injection を待ってから、その行の `!!js` 設定を inject されたコンテキストに対して評価します。

## GitHub からのインストール: ビルドスクリプトの落とし穴

レジストリへの公開は必須ではありません。ユーザーは git ホストから直接インストールできます:

```sh
dsh plugin --profile demo add github:you/hello-plugin
```

しかし git インストールは**ソースを取得し、ビルド済み成果物は取得しません**。`build` スクリプトは実行されないため、TypeScript パッケージは `lib/` 出力なしで届き、読み込みに失敗します。両側で1つずつ、2つのことが必要です:

- **作者**は `prepare` スクリプトを同梱します。pnpm が git インストール後に実行し、公開エントリポイントをソースからビルドします。自己完結している必要があり、兄弟モノレポのチェックアウトのような開発専用コンテキストを前提にしてはいけません。専用の tsdown 設定を使えば、プロジェクト参照や型チェックなしで `src/` をトランスパイルできます。
- **ユーザー**はビルドを許可リストに追加します。pnpm ≥10 は明示的に許可されるまで git 依存関係の `prepare` スクリプトを実行しないため、最初の `add` は失敗します。`dsh` は修正方法を示します。pnpm が表示した正確なパッケージキーを profile の `pnpm-workspace.yaml` にコピーします:

  ```yaml
  allowBuilds:
    dsh-hello-plugin: true
  ```

  そして `add` を再実行します。

この許可は、agent が実行するどのサンドボックスよりも外側で、**インストール時にあなたのマシン上でパッケージのコードを実行する権限**として扱ってください。ソースを信頼するパッケージだけを許可し、コミットを固定します（`github:you/hello-plugin#<sha>`）。そうすれば、後の push が実行内容を黙って変更できません。

ユーザーに許可を求めたくない場合は、代わりにビルド済み成果物を配布します。どちらの形式もビルド権限を必要としません:

- **npm に公開**し、`pnpm publish` 時に `lib/` をビルドします。`dsh plugin add your-package` はビルド済みコードをインストールします。
- **tarball を配布**し、`pnpm pack` で作成します。ユーザーは `dsh plugin add ./hello-plugin-0.1.0.tgz` を実行します。

## 次のステップ

- [plugin とライフサイクル](../framework/index.md) — plugin の完全なライフサイクル
- [CLI 動作リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/cli/reference/README.md) — 正確なレイヤー優先順位、フラグ、profile の仕組み
