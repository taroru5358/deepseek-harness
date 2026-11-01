# Vendored パッケージの rescope

日本語 | [English](../../docs/rescope.md) | [中文](../../docs/rescope.zh.md)

> これは非公式の日本語訳です。英語版（[docs/rescope.md](../../docs/rescope.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

Cordis フレームワークとその基盤ライブラリは [`vendor/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/vendor/README.md) に vendored され、`@deepseek-ai` スコープで公開されます。すべての harness パッケージがフレームワークを peer 依存として宣言しているためです。harness を公開すればこの層も一緒に公開され、上流の名前のままだと、その公開がレジストリ上でそれらの名前を先取りしてしまいます。このページは名前の対応表です。決定とその帰結は [rescope の Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/archived/process/2026-08-10-vendor-package-rescope.md) に、上流のコミットは [`vendor/README.md`](https://github.com/deepseek-ai/deepseek-harness/blob/master/vendor/README.md) にあります。

## 名前の対応

| ディレクトリ | 上流の名前 | 公開名 | 上流バージョン | 役割 |
|---|---|---|---|---|
| `vendor/cordis/` | `cordis` | `@deepseek-ai/cordis` | 4.0.0-rc.11 | フレームワークの中核: `Context`、`Service`、`Fiber`、イベント |
| `vendor/cosmokit/` | `cosmokit` | `@deepseek-ai/cosmokit` | 1.8.1 | フレームワークと Schemastery が基盤とする共有ユーティリティ |
| `vendor/schemastery/` | `schemastery` | `@deepseek-ai/schemastery` | 3.18.0 | すべてのプラグインの `Config` の背後にある設定スキーマ（`Schema`） |
| `vendor/loader/` | `@cordisjs/plugin-loader` | `@deepseek-ai/cordis-plugin-loader` | 1.0.0-rc.5 | `cordis.yml` のロード、プラグイン解決、リポジトリ・キャッシュ |
| `vendor/include/` | `@cordisjs/plugin-include` | `@deepseek-ai/cordis-plugin-include` | 1.0.4 | 設定の include と patch オーバーレイ |
| `vendor/group/` | `@cordisjs/plugin-group` | `@deepseek-ai/cordis-plugin-group` | 1.0.0 | ネストしたプラグイン・グループ |
| `vendor/timer/` | `@cordisjs/plugin-timer` | `@deepseek-ai/cordis-plugin-timer` | 1.1.4 | `ctx` 上の disposal 対応タイマー |
| `vendor/hmr/` | `@cordisjs/plugin-hmr` | `@deepseek-ai/cordis-plugin-hmr` | 1.0.15 | プラグインと設定のホット・モジュール置換 |
| `vendor/logger-console/` | `@cordisjs/plugin-logger-console` | `@deepseek-ai/cordis-plugin-logger-console` | 1.0.0 | コンソール・ロガーのエクスポーター |

サブパスのエクスポートはパスを保ちます。`@cordisjs/plugin-loader/repository` は `@deepseek-ai/cordis-plugin-loader/repository` になります。

## リネームが触れないもの

- **ディレクトリ名と上流ソースのバージョン。** `vendor/hmr/` は `vendor/hmr/` のままで、表は固定したソース・スナップショットの上流バージョンを記録します。そのため manifest は上流のスナップショットとして読めます。vendored な `package.json` 自身の `version` フィールドは harness のリリース済み manifest バージョンで、`pnpm run release:vendor` がそれを上げ、再同期が上流バージョンに戻します。
- **依存関係の範囲。** リネームは範囲を変えずに依存キーを変えます。ワークスペースの manifest は `workspace:` プロトコルを使います。[リポジトリのルール](../../AGENTS.md#conventions)が、正確な DSH 参照と vendor／ネイティブのチルダ範囲を区別します。
- **Loader の `cordis:` 組み込みプレフィックス。** `cordis:include` と `cordis:group` はプロトコルのプレフィックスであり、パッケージ名ではありません。
- **`cordis.yml` 設定ファミリー**。`*.cordis.yml`、`*.cordis.snapshot.yml`、`cordis.patch.yml` を含みます。
- **その語を名前に含む harness パッケージ**。たとえば `@deepseek-ai/dsh-tool-cordis` です。
- **上流のランタイム識別子**。たとえば Schemastery の `Symbol.for('schemastery')` とその `vendor:` メタデータ・フィールドです。
- **製品の識別子。** `GENERIC_SKIPS` 内のファイルは、列挙された上流名について、プリセット ID、イベント名、ディレクトリ・グループ名、ロケールや入力ソースのキーを保ちます。ソースの import、export、モジュール宣言、JSDoc の import 型は、JavaScript／TypeScript のフェンス内も含めて双方向に変わります。YAML プラグインの `name:` 値も変わります。列挙された名前のその他の文字列用途はそのまま残り、他のライブラリ名は引き続き検査されます。
- **書き換え用フィクスチャ。** `scripts/rescope-vendor.spec.ts` と `scripts/gen-third-party-notices.spec.ts` は、両方のパッケージ表記をテスト・データとして保ちます。
- **`docs/` の外の散文。** `vendor/*/README.md`、パッケージの README、Agent Notes は書かれたときの名前を保ちます。そこでの素の `cordis` は、Python SDK のオプション名や agent プリセットの ID であることもあります。`docs/` の内側では、散文と Markdown のフェンスは、上記の監査済み識別子を除いてトークンのルールに従います。

## 自分のコードで変更が必要な箇所

| 箇所 | 変更前 | 変更後 |
|---|---|---|
| モジュールの import | `import { Context } from 'cordis'` | `import { Context } from '@deepseek-ai/cordis'` |
| 型付きイベントのマージ | `declare module 'cordis'` | `declare module '@deepseek-ai/cordis'` |
| `package.json` の依存キー | `"@cordisjs/plugin-hmr": "^1.0.15"` | `"@deepseek-ai/cordis-plugin-hmr": "^1.0.15"` |
| `cordis.yml` のプラグイン・エントリー | `name: '@cordisjs/plugin-include'` | `name: '@deepseek-ai/cordis-plugin-include'` |

## 適用、検証、巻き戻し

[`scripts/rescope-vendor.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/rescope-vendor.ts) が上記の対応を所有し、リネームを実行します。そのため参照を手でリネームすることはありません。

```sh
pnpm run rescope-vendor            # report what would change
pnpm run rescope-vendor --apply    # rewrite every reference
pnpm run rescope-vendor:check      # assert the post-state; runs in the hygiene gate
pnpm run rescope-vendor --apply --reverse   # return to the upstream names
```

上流との同期の後に再適用し（[手順](https://github.com/deepseek-ai/deepseek-harness/blob/master/vendor/README.md)）、それが出力する再生成に従います。ロックファイルには `pnpm install`、`pnpm run gen-third-party-notices`、触れたバイリンガル・ペアには `pnpm run verify-translation-pairing --write` です。
