# 1. 最初のプラグイン

日本語 | [English](../../../docs/cordis-tutorial/01-first-plugin.md) | [中文](../../../docs/cordis-tutorial/01-first-plugin.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cordis-tutorial/01-first-plugin.md](../../../docs/cordis-tutorial/01-first-plugin.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

ここで使うローダー構成では、Cordis プラグイン・モジュールは `apply` 関数を名前付き export します。Cordis がそれをロードすると、**context** — プラグインが提供するすべてを登録する `ctx` オブジェクト — を伴って `apply` を呼びます。

## プラグインを書く

`tmp/cordis-tutorial` ディレクトリ（[セットアップ](index.md#setup)を参照）に `hello.ts` を作成します:

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'hello'

export function apply(ctx: Context) {
  console.log('hello from my first plugin')
}
```

`name` export は任意の表示メタデータで、診断でプラグインにラベルを付けます。

## アプリを構成する

このチュートリアルのランチャーは設定からアプリケーションを組み立てます。`cordis.yml` を作成します:

```yaml
- name: './hello.ts'
```

このファイルはプラグイン・エントリーのリストです。`name` はモジュール指定子 — 相対パスまたは npm パッケージ名 — で、ローダーはすべてのエントリーをマウントします。エントリーは並行して開始するため、リスト上の位置はどのプラグインが最初にロードされるかを保証しません。順序はファイル内の位置ではなく、サービスの依存関係（`inject`、[第3章](03-services.md)）から来ます。

## 実行する

```sh
node --import tsx ../../vendor/cordis/bin.js
```

期待される出力:

```
hello from my first plugin
```

何も実行中でなくなると、プロセスは自動的に終了します。何が起きたか:

1. ランチャーがルート `Context` を作成し、**Loader** プラグインをマウントしました。
2. Loader が `cordis.yml` を読み、`./hello.ts` を解決し、子プラグインとしてマウントしました。
3. Cordis があなたの `apply(ctx)` を呼びました。

あなたのファイルにフレームワークのブートストラップ・コードはありません。プラグインは提供するものを記述し、`cordis.yml` がアプリケーションを構成します。たとえば [`dsh` の base](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/base/cordis.patch.yml) は、デプロイ用オーバーレイがパッチする、より長いプラグイン構成です。

## 他の2つのプラグインの形

関数が最も一般的な形ですが、Cordis は3つを受け付けます:

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

// 1. Function plugin (what you just wrote).
export function apply(ctx: Context) {}

// 2. Object plugin: an object with an `apply` method.
export const objectPlugin = {
  name: 'object-plugin',
  apply(ctx: Context) {},
}

// 3. Class plugin: a Service subclass (covered in chapter 3).
export class MyService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'myTutorialService')
  }
}
```

サービスを公開する必要が生じるまでは関数形式を使ってください。クラス形式がその場所を得るのはどんなときかは[第3章](03-services.md)で扱います。

## 壊してみる

`apply` を throw させます:

```ts ignore-check
export function apply(ctx: Context) {
  throw new Error('apply exploded')
}
```

もう一度実行します。プロセスはあなたのエラーで終了します。ロードに失敗するプラグインは、スキップされたエントリーではなく、目立つ失敗です。

早めに知っておく価値のある注意点が1つあります。モジュールを**解決**できない設定エントリー — パスやパッケージ名のタイポ — は、プロセスをクラッシュさせるのではなく Cordis ロガー・サービスを通じて報告されます。起動時には、コンソール・エクスポーターが監視する前にその報告が失われることがあります。新しく追加したエントリーが何もしていないように見える場合は、まず綴りを確認してください。

次: [ライフサイクルと効果](02-lifecycle-and-effects.md) — プラグインがアンロードされるときに何が起きるか。

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
