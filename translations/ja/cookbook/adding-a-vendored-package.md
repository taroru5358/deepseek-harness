# クックブック: vendored パッケージの追加

日本語 | [English](../../../docs/cookbook/adding-a-vendored-package.md) | [中文](../../../docs/cookbook/adding-a-vendored-package.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cookbook/adding-a-vendored-package.md](../../../docs/cookbook/adding-a-vendored-package.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

harness が別の上流 Cordis パッケージ（例: `@cordisjs/plugin-http`）を必要とする場合、npm 依存として追加するのではなく、`vendor/` 以下の固定ソースとして **vendored** にします。[vendor/README.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/vendor/README.md) が理由を述べ、すでに vendored なパッケージの*更新*を扱います。このガイドは**新しい**パッケージを追加するためのファイル単位のチェックリストです。（既存の vendored セットに対して検証済みです。乖離した場合はここを修正してください。）

## 1. ソースをコピーする

```
vendor/<dir>/
  package.json     # from upstream; rescope the name, keep exports/type (publishable release member, no private flag)
  tsconfig.json    # extends ../../tsconfig.base.json (see configuration below)
  src/             # the upstream src/ verbatim
  README.md LICENSE # if upstream ships them
```

`tsconfig.json` は他の vendored パッケージを反映します。`rootDir: src`、`outDir: lib/types`、上流コードが必要とする厳格さの緩和、および import する他の各 vendored パッケージの `references` エントリーです:

```jsonc
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src", "outDir": "lib/types",
    "noUncheckedIndexedAccess": false, "exactOptionalPropertyTypes": false,
    "noImplicitOverride": false, "noUnusedLocals": false, "noUnusedParameters": false
  },
  "include": ["src"],
  "references": [{ "path": "../cordis" }, { "path": "../cosmokit" }]
}
```

`package.json` の不変条件: 上流の `exports`/`type` を保ちながら `name` を再スコープし（[マッピング](../rescope.md)）、宣言メタデータを `lib/types` に向け、`.d.ts` と `.d.ts.map` の宣言出力を公開し、cordis 依存を `peerDependencies` に列挙します（上流マニフェストと一致させる）。vendored パッケージは公開可能なリリース・メンバーであるため、`private: true` を設定してはならず、`publishConfig.access: public` を設定しなければなりません。`version` フィールドは harness のリリース順序に従います（[vendor/README.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/vendor/README.md) を参照）。上流の推移的依存は、それ自体が vendored であるか既に存在していなければなりません。1つのパッケージを vendoring することは、しばしばその依存ツリーを vendoring することを意味します（例: `@cordisjs/plugin-http` は `@cordisjs/fetch-file` を引き込みます）。

vendored TypeScript ソース内のローカル相対 import／export は、コピー後に明示的な `.ts` 指定子を使います。これは上流とのリポジトリ・ローカルのビルド差分です。`rewriteRelativeImportExtensions` は実行時の import を `.js` として出力し、宣言は NodeNext/Node16 の TypeScript コンシューマーが解決できる明示的な `.ts` 指定子を保ちます。

## 2. ルート設定に登録する

| ファイル | 変更 |
|---|---|
| `tsconfig.base.json` | `paths` に `"<npm-name>": ["./vendor/<dir>/src"]` を追加する |
| `tsconfig.host.json` | `references` に `{ "path": "./vendor/<dir>" }` を追加する（`packages/*` エントリーより前。vendored コードは host 集約を通じてのみグラフに入る） |
| `vendor/README.md` | マニフェスト表の行（dir、npm 名、バージョン、上流リポジトリ、コミット SHA）を追加し、ローカル変更を記録する |
| `scripts/publint-all.ts` | vendored パッケージ自体がここから公開される場合のみ（vendored 依存は通常そうではない — スキップ） |

glob によって自動的にカバーされ、編集は不要です: ルート `package.json` の workspaces（`vendor/*`）、`tsdown.config.ts`、`vitest.config.ts`、`.oxlintrc.json`。パッケージごとの `vendor/<dir>/tsdown.config.ts` は、ビルド構成がルート既定と異なる場合（デュアル ESM/CJS または複数エントリー — `vendor/schemastery` と `vendor/logger-console` を参照）にのみ必要です。そのエントリーは `lib/types` 以下に出力された JS を読むべきです。

## 3. マニフェスト・ガードに注意する

`scripts/check-vendor-manifest.sh`（pre-commit フック）は、`vendor/README.md` もステージされずに `vendor/*/src` 以下の何かがステージされると失敗します。コミットが通るように、マニフェスト更新をソースと一緒にステージしてください。

## 4. 検証する

```sh
pnpm install        # registers the workspace
pnpm run typecheck
pnpm run build && pnpm run constraints
```

[テスト方針](../testing.md)が選択する振る舞いチェックを実行します。ソースの `paths` マップは `tsconfig.base.json` に一度だけ存在し、すべてのグラフにサービスします。重要な分離境界はプロジェクト参照グラフです。vendored ソースは独自の `vendor/<dir>/tsconfig.json` を通じて参照されなければならず、集約の strict プログラムに引き込まれてはいけません（[レイアウト](../development.md#typescript-project-layout)）。
