# パッケージ・サーフェスのルール

日本語 | [English](../../../docs/defensive-patterns/package-surface.md) | [中文](../../../docs/defensive-patterns/package-surface.zh.md)

> これは非公式の日本語訳です。英語版（[docs/defensive-patterns/package-surface.md](../../../docs/defensive-patterns/package-surface.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

パッケージが何をエクスポートし、どうビルドされ、依存関係をどう宣言するかに関するルールです。[development.md](../development.md#typescript-project-layout) がレイアウト自体を説明し、[packages/AGENTS.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/AGENTS.md) が常設の指示を担います。

## プラグイン・パッケージは Loader がロードするものをエクスポートする

プラグイン・パッケージのエントリー・ポイントは 2 つの形式のいずれかを取り、混在させません。**関数プラグイン**は `name`、`inject`、`Config`、`apply` を名前付きエクスポートします。`name`、`inject`、`apply` は小文字で、`Config` は大文字で、`interface` としても Schemastery スキーマ値としても現れます。**サービス・プラグイン**はサービス・クラスを default エクスポートし、Loader はそのようにロードします。4 つの隣に default エクスポートがあると、Loader は関数プラグインの名前空間を破棄します。それら以外に、Host エントリーは、他のパッケージが拡張したり throw したりする Service Definition クラスとエラー型をエクスポートしてよく、ケイパビリティ・シームはまさにそれらを消費します。テストの便宜のための値をエクスポートすることは決してありません。UI プラグイン・パッケージの `/client` エントリーはより厳格で、[packages/client/AGENTS.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/AGENTS.md) がその規律を所有します。同一パッケージのテストは相対 `../src/...` パスまたは `./src/*` サブパスを通じて内部に到達するため、テスト専用の必要が恒久的なエクスポートになることはありません。このルールは、テストの便宜が公開 API に固定されるのを防ぎます。

- Loader の 4 つとその正確な形式は [cordis-primer.md](../cordis-primer.md#loader-configuration) で定義されています。
- 判定条件: 関数プラグインのエントリー（`src/index.ts`）が 4 つを超えるエンティティをエクスポートする。4 つの隣に default エクスポートが追加される、または 2 つの形式が混在する。`/client` エントリーが実装値をエクスポートする、または別の機能プラグインに実装値を import させる。フィクスチャが相対パスで別のパッケージに到達する。
- 境界: 制約されるのは Loader プラグインのエントリー・ポイントだけです。内部モジュール、ヘルパー関数、フックは値をエクスポートし続け、Host エントリーの Service Definition クラスとエラー型は期待されるエクスポートです。また、テスト・フィクスチャは自身のパッケージ内に留まります。別のパッケージの `src` への相対パスは `exports` マップとパッケージ境界のチェックを迂回します。
- 例外: 既定ではありません。真に必要な例外にはドメイン所有者の確認を要します。

準拠したエントリーは 4 つのエクスポートを持ち、同一パッケージのテスト・パスは 5 つ目のエクスポートではなくコメントとして記録されます。

```ts
import type { Context } from '@deepseek-ai/cordis'
import z from '@deepseek-ai/schemastery'

// The plugin entry: the Loader four, and no fifth value
export const name = 'web-app'
export const inject = ['webServer']
export interface Config { /* … */ }
export const Config = z.object({ /* … */ })
export function apply(ctx: Context, config: Config): void { /* … */ }

// A same-package test imports '../src/internal/header.ts' instead of this entry
```

エントリーが持つべきでないエクスポートを得る 3 つの方法で、いずれも diff で見えます。

```ts ignore-check
// The entry widened so a test could reach one helper
export { isHeaderLine } from './internal/header.ts'

// Both plugin forms on one entry: the Loader discards the four
export default class WebApp {}

// A fixture reaching into another package by relative path
import { isSurfaceEligibleType } from '../../core/session/src/surface.ts'
```

## パッケージ形状は 2 つあり、どちらも動的プラグイン・パッケージである

新しいパッケージは Host パッケージか Host + Client パッケージのどちらかであり、どちらも動的プラグイン・パッケージです。第 3 の形式はなく、例外もありません。シェルが [packages/client/web/src/platform.ts](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/web/src/platform.ts) の `PLATFORM_MODULES` を通じてシードする静的クライアント・ライブラリ（`client-store`、`ui-slots`、`ui-primitives`、`ui-dockkit`）と、[packages/util/](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/util) 配下のゼロ依存ライブラリは、このルール以前から存在し、クローズ済みです。それらは現状のまま動作し続け、新しいパッケージが加わることはありません。真に両面を持つパッケージだけがコンパイラー・フェースを分割し、ビルド出力は `lib` に置かれ、`include`、`external`、roster のリストは手書きではなくワークスペースをスキャンして導出されます。手書きのリストや発明されたコンパイラー・フェースはビルド出力とマニフェストを食い違わせ、後続のすべてのパッケージが独自のリストを保守しなければならなくなります。

- 判定条件: 新しいパッケージが動的プラグイン・パッケージ以外（静的リンクのバンドル、手書きの全 external リスト）としてビルドされる。第 3 の `tsconfig` バリアントがある。`include`、`external`、roster のリストが手書きである。
- ここの数値を信じるのではなく、形状を再計算してください。`packages/*/*` 配下の `tsconfig.json` を列挙し、次に `tsconfig.client.json` と `tsconfig.host.json` も持つパッケージ、次に `dsh.client` を宣言するパッケージを列挙します。
- 境界: 両面パッケージは `include` または `exclude` に現れてかまいません。
- 例外: ありません。承認経路もありません。パッケージの形状やコンパイラー・フェースの変更は所有する文書から始まり、ドメイン所有者の確認を要します。
- 所有者: コンパイラー・フェースは [development.md](../development.md#typescript-project-layout)、パッケージ・グループは [packages/README.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/README.md)、手順は [adding-a-package.md](../cookbook/adding-a-package.md)。

2 つの形状のいずれかにあるパッケージは、標準のマニフェスト・フィールドを持ち、手書きの external リストを持ちません。

```json
{
  "name": "@deepseek-ai/dsh-example",
  "type": "module",
  "main": "lib/index.js",
  "types": "lib/types/index.d.ts",
  "exports": {
    ".": { "types": "./lib/types/index.d.ts", "default": "./lib/index.js" }
  }
}
```

手書きの `external` リストと、その隣に 1 つのパッケージのために発明された `tsconfig.foo.json` は、スキャンに何も導出させず、ビルド出力とマニフェストが一致しなくなります。

```json
{ "name": "@deepseek-ai/dsh-example", "external": ["@deepseek-ai/*"] }
```

## `package.json` の `exports` は標準キー・セットを使う

`package.json` の `exports` マップは標準キー・セットだけを使い、各キーはリポジトリ全体ですでに 1 つの意味を持ち、パッケージが自身のローカルな目的のためにキーを開くことはありません。文書化する所有者なしにごく少数のパッケージだけが使うキーは前例ではありません。キーの正式化（下記の未文書キーを含む）は、所有する文書とチェックを同じ変更で変えることを意味します。小さな共有キー・セットがあるからこそ、コンシューマーは名前からキーの目的を推測でき、スキャンはエクスポート・マップを判定できます。

- 標準キーはそれぞれリポジトリ全体で 1 つの意味を持ち、`packages/*/*/package.json` と `apps/*/package.json` にまたがります。`.`（パッケージ・エントリー）、`./package.json`（ツールチェーン・アクセスであり、セマンティック・エクスポートではない）、`./src/*`（テストとツールのためのソース・パススルーで、その文書化状況はまだ未決）、`./client`（ブラウザ・エントリーで、`dsh.client` 宣言と対になる）、`./types`（型エントリー）、`./typert`（typert 型ランタイム）、`./remote`（`@Remote` 呼び出しサーフェス）、`./locale/*.json`（ロケール・リソース）、`./cordis.patch.yml`（`dsh.bundle` が宣言するバンドル・パッチ・レイヤー）。件数はここに記載しません。次のパッケージで陳腐化するからです。下の再計算コマンドがそれらを出力します。
- そのセット外のキー（5 パッケージの `./brand`、4 パッケージの `./worker`、各 2 パッケージの `./startup`、`./runner`、`./protocol`、および多数の一回限りのキー）には文書化された意味がありません。それらはレビュー待ちであり、コピーすべきパターンではありません。
- `dsh-tools` はさらに、再利用可能な呼び出しステージとバッチ・スケジューリングのために `./pipeline` を公開します。その main バンドルと pipeline バンドルは、エラー・クラスの同一性を保つためにランタイム・チャンクを共有します。ワークスペースの制約がエクスポート・ターゲットと公開ファイルをチェックします。
- 判定条件: 変更が標準セット外のエクスポート・キーを追加するか。
- 例外: 既定ではありません。新しいキーはまず所有する文書とチェックを変え、ドメイン所有者の確認を伴います。
- 分布を再計算します:

```sh
python3 -c "import json,glob,collections; c=collections.Counter(); [c.update(json.load(open(p)).get('exports',{}).keys()) for p in glob.glob('packages/*/*/package.json')+glob.glob('apps/*/package.json') if isinstance(json.load(open(p)).get('exports'),dict)]; print(c.most_common())"
```

準拠したマップは、リポジトリ全体の意味で使われた 3 つの標準キーを持ち、同じパッケージが自身の client エントリーも宣言しています。

```json
{
  "exports": {
    ".": { "types": "./lib/types/index.d.ts", "default": "./lib/index.js" },
    "./client": { "types": "./lib/types/client/index.d.ts", "default": "./lib/client.js" },
    "./types": { "types": "./lib/types/types.d.ts", "default": "./lib/types/types.js" },
    "./src/*": "./src/*",
    "./package.json": "./package.json"
  }
}
```

1 つのパッケージのローカルな目的のために開かれたキー（ここでは `./brand` と `./worker` で、その文書化状況は未決）は、エクスポート・マップをコンシューマーもスキャンも読めないパッケージごとの慣習に変えます。

```json
{
  "exports": {
    ".": "./lib/index.js",
    "./brand": "./lib/brand.js",
    "./worker": "./lib/worker.js"
  }
}
```

## 依存関係の宣言とバージョンの一貫性は依存関係スキャンによって強制される

依存関係の宣言とバージョンの一貫性は、依存関係スキャンによって機械的に保証されます。このページは `devDependencies` と `peerDependencies` のフィールド・ルールも、独自のパッケージ受け入れリストも繰り返しません。スキャンが失敗したら、ローカルで再現し、それが名指ししたマニフェストを修正します。スキャンとそのポリシー・ファイルがルールを所有するからです。変更を着地させるためにスキャンを編集したりポリシーを緩めたりすることは通常の編集ではなく、ドメイン所有者の確認が必要です。

- エントリー・ポイント: [verify-package-dependencies.ts](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/verify-package-dependencies.ts)。`pnpm run verify-package-dependencies` として、`scripts/run-gates.ts` からは `package-dependencies` ゲートとして、`.github/workflows/release.yml` からは直接実行されます。
- ポリシーの所有者: [package-dependency-policy.ts](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/package-dependency-policy.ts) が `PACKAGE_DEPENDENCY_POLICY` をエクスポートします。宣言セクション、バージョンの形式、免除はそこで定義され、パッケージ・レベルの文脈は [packages/README.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/README.md) にあります。
- 境界: このページはパッケージ、フィールド・ルール、しきい値を列挙しません。代わりに所有するファイルを指します。
- 配線を見つける: `grep -rn "verify-package-dependencies" package.json .github/workflows scripts/run-gates.ts`。
- 例外: スキャンまたはポリシーへの変更にはドメイン所有者の確認と、変更に記録された理由が必要です。

再現可能な手順は、再現し、診断を読み、それが名指ししたマニフェストを修正することです。

```sh
# 1. Reproduce what CI reports
pnpm run verify-package-dependencies

# 2. The diagnostic names the manifest, the section, the dependency, and the expected range
#    packages/example/package.json: dependencies.@deepseek-ai/dsh-base must use workspace:*, found workspace:~

# 3. The fix goes into the manifest, never into the policy file or the scan
#    "dependencies": { "@deepseek-ai/dsh-base": "workspace:*" }

# 4. Re-run the scan until every package matches the published policy
pnpm run verify-package-dependencies
```

機械的な保証を失う 2 つの方法は、マニフェストではなくチェックを変えること、またはここでルールを再掲して所有者から乖離させることです。

```text
Relaxing PACKAGE_DEPENDENCY_POLICY, or dropping the scanned section that fails, to go green
Restating a required-devDependency package list or the field rules on this page instead of linking the policy file
```

## 内部依存関係の構文はパッケージ・ファミリーに従う

内部依存関係の構文はパッケージ・ファミリーに従います。`@deepseek-ai/dsh` と `@deepseek-ai/dsh-*` は `workspace:*` を使い、その他のすべての内部パッケージは `workspace:~` を使い、`workspace:^` は廃止されました。同じ依存関係名はすべてのワークスペースで同じように書かれ、不一致はレビュアーが免除できる警告ではなく、スキャンが無条件に失敗する原因になります。統一された構文は 1 つの解決経路を保ち、依存関係の重複インスタンスと、それに続く解決の差異を防ぎます。

- [verify-package-dependencies.ts](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/verify-package-dependencies.ts) の `workspaceRange()` で判定します。`@deepseek-ai/dsh` と `@deepseek-ai/dsh-*` のファミリーは `workspace:*` を、その他のすべての内部パッケージは `workspace:~` を得ます。スキャンは管理対象のマニフェストを走査し、同じ依存関係名のすべての宣言を比較します。
- 境界: ワークスペース・メンバーではない同一名前空間のパッケージは通常のサードパーティ範囲を宣言し、ファミリー・ルールの外にあります。`@deepseek-ai/libreoffice-kit` は `apps/desktop-host/package.json`（dependencies）、`packages/bundle/web-app/package.json`（devDependencies）、`packages/document/office-to-pdf/package.json`（dependencies）、`packages/skill/skill-office/package.json`（dependencies）で `^0.1.5` と宣言されています。
- 分布はここで再掲しません。下の再計算コマンドが `workspace:*`、`workspace:~`、`workspace:^`、およびファミリー・ルール外の外部範囲宣言の現在の件数を出力します。
- 例外: スキャンが機械的に決定します。真の例外は、ドメイン所有者の確認と変更上の理由を伴ってスキャンとポリシー・ファイルを変えることを意味します。
- 再計算:

```sh
python3 -c "import json,glob,collections; c=collections.Counter(); [c.update([r]) for p in glob.glob('apps/*/package.json')+glob.glob('packages/*/*/package.json')+glob.glob('vendor/*/package.json') for s in ('dependencies','devDependencies','optionalDependencies','peerDependencies') for n,r in (json.load(open(p)).get(s) or {}).items() if n.startswith('@deepseek-ai/')]; print(c.most_common())"
```

各ファミリーをその形式で示し、非メンバーの例外は通常のバージョン範囲のままにします。

```json
{
  "dependencies": {
    "@deepseek-ai/dsh-base": "workspace:*",
    "@deepseek-ai/cordis": "workspace:~",
    "@deepseek-ai/libreoffice-kit": "^0.1.5"
  },
  "devDependencies": {
    "@deepseek-ai/dsh-typert-registry": "workspace:*"
  }
}
```

`workspace:~` を使う `dsh-*` パッケージと、外部範囲に固定されたワークスペース・メンバーは、それぞれスキャンが無条件に失敗するケースであり、廃止された完全一致バージョン固定のルールを代替として使うことはできません。

```json
{
  "dependencies": {
    "@deepseek-ai/dsh-base": "workspace:~",
    "@deepseek-ai/cordis": "^4.0.0"
  }
}
```

## 新しい resolve フックは完全に拒否される

新しい `resolve` フックは既定で拒否されます。確固たる禁止であり、できれば避けたいという選好ではありません。真の必要には 2 つの前提条件が同時に必要です。変更の動機となった指定子にパッチを当てるのではなく、解決戦略を全体として再監査すること、そして何かが着地する前のドメイン所有者の確認です。このケースは特別だという著者自身の判断は確認ではありません。

- 判定条件: 変更が `resolve` フックを追加するか。同じ指定子を書き換える既存フックのラッパーも含みます。
- 境界: このルールはフックを追加してよいかどうかを決めます。解決、ホット・リロード、プラグイン管理の責任の分離は別に統治され、新しいフックを許可しません。
- 唯一の解決経路は、`dsh` のソース起動が登録する tsx ESM フックで、その根拠は [source-launch の Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-07-29-dsh-source-launch-tsx-esm.md) にあります。
- 例外: 上記の 2 つの前提条件だけであり、両方とも変更に記録されます。

新しい解決の振る舞いが真に必要な場合の準拠経路を、順に示します。

```text
A new resolve hook is allowed only after both preconditions hold:

1. Re-audit the resolve strategy as a whole — resolution, hot reload, and plugin
   management responsibilities together, not only the specifier at hand.
2. Record the domain owner's confirmation on the change.
3. Only then add the hook, linking both records from the change.

Until both hold, the answer is no; "this case is special" is the author's judgment,
not a confirmation.
```

1 つの指定子がこのパッケージの期待する場所に着地するように追加されたフック、または既存フックの同等のラッパーでは、解決経路に分岐が増え、リゾルバーが返すものが `node_modules` と一致しなくなります。

```ts ignore-check
// Added so one specifier resolves to the local source copy
export async function resolve(specifier: string, parentURL: string, importAttributes: object) {
  if (specifier === '@deepseek-ai/dsh-example') return { url: localSourceUrl, shortCircuit: true }
  return defaultResolve(specifier, parentURL, importAttributes)
}

// The equivalent shape: wrapping the hook that already exists
const previous = hooks.resolve
hooks.resolve = (specifier, context, next) =>
  pinned.has(specifier) ? pinned.get(specifier) : previous(specifier, context, next)
```

## パッケージ・モジュールはモジュール・スコープで await しない

パッケージのソース・モジュールは、自身の非同期処理を、自身の拒否を処理する明示的な `main()` のような、自身が呼ぶ関数の内部で開始し、モジュール・スコープで `await`、`for await`、`await using` を決して使いません。モジュール・スコープの待機はすべてのインポーターの評価を待たせ、拒否をモジュール評価の失敗に変え、ブラウザ・ワーカー・イメージの同期的な CommonJS ラッパーへ低レベル化できません。

- [verify-no-top-level-await.ts](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/verify-no-top-level-await.ts) で判定され、`pnpm run verify-no-top-level-await` として、また `scripts/run-gates.ts` からは `no-top-level-await` ゲートとして実行されます。これは `packages/*/*/src/**/*.{ts,tsx}` のすべてのファイルを解析し、実行可能ファイルを宣言によってのみ除外します。`#!` で始まるソース、またはパッケージの `bin` フィールドが名指しする `lib/` 出力のソースです。
- 例外: ありません。
