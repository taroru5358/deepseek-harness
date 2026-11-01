# クックブック: ライブ設定フォーム

日本語 | [English](../../../docs/cookbook/adding-a-settings-card.md) | [中文](../../../docs/cookbook/adding-a-settings-card.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cookbook/adding-a-settings-card.md](../../../docs/cookbook/adding-a-settings-card.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

プラグインの Config スキーマでライブ・フィールドを宣言し、製品所有の設定カードを通じて公開します。export される `Config` インターフェースは、各 `Volatile<T>` 参照を含め、プラグインが受け取る値を記述します。

## 1. ライブ・フィールドを宣言する

```ts
import type { Context, Volatile } from '@deepseek-ai/cordis'
import z from '@deepseek-ai/schemastery'

export interface Config {
  endpoint: Volatile<string | undefined>
  retries: Volatile<number>
}

export const Config = z.object({
  endpoint: z.string().volatile(),
  retries: z.number().step(1).min(0).default(3).volatile(),
})

export function apply(ctx: Context, config: Config): void {
  ctx.on('loader/volatile-update', () => {
    ctx.logger.info('Retry limit: %d', config.retries.get())
  })
}
```

操作を開始するときに `.get()` を読みます。一貫したスナップショットを必要とするリクエストは、値を一度だけ取得します。フィールド間の Config 検証には `.check()` を使います。これらのチェックは永続化前に Host で実行され、シリアライズされたフォーム・スキーマからは省略されます。

## 2. プラグインを構成する

各インスタンスに一意のプロファイル・エントリー id を付けます。base バンドルは settings と config-editor をマウントします。カスタム・プロファイルでは、同じサービスをマウントする前にそのパッケージの README を参照してください。

`role('secret')` は値をフォーム応答から除外します。認証情報ドメインが管理する値には認証情報参照を使います。通常のフィールドは設定スキーマに入れません。

## 3. 編集を検証する

Plugins でフィールドを変更して保存します。プロファイル・パッチ、次のコンシューマー操作、プラグイン・インスタンス同一性が変わらないこと、再起動後の復元を検証します。無効な値を拒否し、ファイルもライブ値も変わっていないことを検証します。

カスタム・プラグイン・ページは、Plugins ページ所有者から `form.state` と `form.mutate(operations, expectedRevision)` を受け取ります。Host は完全な Config を検証し、ConfigEditor と volatile HMR を通じて編集を適用します。既存のカード例は[プラグイン設定パッケージ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-settings-plugins/README.md)が所有します。

## 4. 別のプラグインのページに貢献する

自分が所有しないバンドル、行、公式プラグインについて何か言いたいプラグインは、`plugins.detail.actions`（ページ先頭のコントロール）、`plugins.detail.badge`（タイトルの隣のタグ）、`plugins.detail.section`（ページ自身のコンテンツの下のセクション）に登録します。各エントリーはページの `subject` — `{ kind: 'bundle', pkg }`、`{ kind: 'row', pkg, row }`、`{ kind: 'item', id }` — とともにレンダリングされ、何もない subject には null を返します:

```tsx ignore-check
ctx.slots.inject('plugins.detail.badge', () => ctx.slots.register({
  name: 'plugins.detail.badge',
  id: 'acme-update',
  locale: 'acmeUpdate',
}, ({ t, subject }) => subject.kind === 'bundle' && hasUpdate(subject.pkg) ? <Tag tone="info">{t('update')}</Tag> : null))
```

## 5. ブラウザ側の半分が乗る場所

ブラウザ側の半分は[クライアント・モジュール・システム](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/client/modules)がページに提供します。これは有効な Loader エントリーを走査して `dsh.client` を宣言するパッケージを見つけ、それぞれのビルド済み `./client` export を提供しますが、パッケージの半分を、指定子が裸のパッケージ名である Loader 行に結び付けます。サブパス export からマウントされた行は決して半分を運びません。そのため、1つのパッケージを複数の行に分割するバンドルは、半分をルート行に保ち、登録するすべてのページはその行が無効になると消えます。他の行より長く存続する必要があるページを持つサブプラグインは、独自のパッケージとして出荷します。

ビルド済みの `./client` ファイルは、クライアント・モジュール・システムの lazy-CJS ファクトリー形式でなければなりません。パッケージ名と、ページのモジュール・ローダーを持つ `factory(require)` を登録する1つのスクリプトで、[クライアント・モジュール・システムの README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/modules/README.md)に説明があります。それを出力する `clientBundle` tsdown プリセットは公開パッケージではなく `packages/client/tsdown.client.ts` にあり、このリポジトリ外のパッケージはそのビルドを自分で再現します。

```jsonc
{
  "exports": {
    ".": { "types": "./lib/types/index.d.ts", "default": "./lib/index.js" },
    "./client": { "types": "./lib/types/client/index.d.ts", "default": "./lib/client.js" }
  },
  "dsh": { "client": { "platform": "web", "inject": ["@deepseek-ai/dsh-client-ui-settings"] } }
}
```

組み込み名前空間のコンパニオン・パッケージは、空の Host `apply` を持つ client-only パッケージ内の同じブラウザ側の半分です。Web 構成のプラグイン名簿（[`packages/bundle/web-app/cordis.patch.yml`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/web-app/cordis.patch.yml)）に列挙され、`ctx.configForms.whileServed` を通じて `plugins.item` に登録するため、Host が名前空間にサービスしている間だけページが存在します。
