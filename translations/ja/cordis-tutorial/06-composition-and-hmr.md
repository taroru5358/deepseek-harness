# 6. 構成と HMR

日本語 | [English](../../../docs/cordis-tutorial/06-composition-and-hmr.md) | [中文](../../../docs/cordis-tutorial/06-composition-and-hmr.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cordis-tutorial/06-composition-and-hmr.md](../../../docs/cordis-tutorial/06-composition-and-hmr.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

これまでに構築したすべてのケイパビリティはプラグインであり、`cordis.yml` がアプリケーションのプラグイン・ツリーを選択します。この章ではその構成を変更し、プラグインをホットリロードし、決してロードされないプラグインを診断します。

## エントリーは名前だけではない

設定エントリーは `name` と `config` 以外のメタデータも受け付けます:

```yaml
- id: greeter          # stable identity for this entry
  name: './greeter.ts'
- id: consumer
  name: './consumer.ts'
  disabled: true       # keep the entry, skip mounting it
```

`id` はエントリーに安定した同一性を与え、ローダーが既存エントリーへの編集と、削除＋追加を区別できるようにします。`disabled: true` はエントリーを削除せずにプラグインをアンマウントします。元に戻せば、プラグイン（およびそのサービスを待つすべての PENDING）が再びロードされます。

グループは、1つの単位としてロード・アンロードされるエントリーのサブリストをネストします。`isolate` はグループにサービス名の独自インスタンスを与えます。2つのグループが、互いに影響せずに異なる設定の `shell` プロバイダーを見ることができます。詳細は [Cordis 入門](../cordis-primer.md)と[サービス分離の例](../user/develop/framework/service.md#service-isolation)で扱います。

## ホット・モジュール置換

アンロードが効果を解放し（[第2章](02-lifecycle-and-effects.md)）、ロードが依存関係に従う（[第3章](03-services.md)）ため、HMR は実行中のプラグインをアンロードしてロードすることで置き換えられます。`@deepseek-ai/dsh-hmr` プラグインはファイルを監視し、保存時にまさにそれを行います。

`tmp/cordis-tutorial` で `cordis.yml` を書きます:

```yaml
- id: logger
  name: '@deepseek-ai/cordis-plugin-logger-console'
- id: timer
  name: '@deepseek-ai/cordis-plugin-timer'
- id: hmr
  name: '@deepseek-ai/dsh-hmr'
  config:
    root: ['.']
- id: hello
  name: './hello.ts'
```

2つのサポート・プラグインがリストに加わりました。HMR は Cordis ロガー・サービスを通じてログを出すため、コンソール・エクスポーターがなければそのメッセージは見えません。また、デバウンスのために `timer` サービスを `inject` します。`@deepseek-ai/cordis-plugin-timer` がないと、静かに永遠に PENDING のままです。その沈黙が次のセクションの主題です。

HMR は Node のローダー内部を、Loader のネイティブ・ヘルパーを通じて読みます。Cordis を tsx の下で実行します:

```sh
node --import tsx ../../vendor/cordis/bin.js
```

次に `hello.ts` を編集し — ログ・メッセージを変えます — 保存します:

```
hello from my first plugin
2026-07-22 15:44:36 [I] hmr watching [ '.' ]
2026-07-22 15:44:39 [I] hmr reload plugin at hello.ts
hello from my EDITED plugin
```

古いインスタンスがアンロードされ（すべての効果が巻き戻され）、新しいコードがロードされ、`apply` が再び実行されました。Ctrl-C でプロセスを停止します。`cordis.yml` 自体の編集も拾われます。ローダーは `id` でエントリーを差分し、変更されたものだけをマウント、アンマウント、再設定します。上のエントリーが明示的な `id` を持つのはそのためです。id のないエントリーは読み取りごとに生成された id を得るため、設定ファイルを編集すると、自身の行が変わっていなくても削除＋追加として扱われ、再マウントされます。

## 決してロードされないプラグインを診断する

依存関係駆動ロードの裏側です。`inject` が誰も提供しないサービスを名指しするプラグインは、何も出力せずに永遠に待ちます。エラーは出ません。プロバイダーが後からマウントされる可能性があるため、PENDING は正当な状態です。

状態を直接見ることができます。すべてのコンテキストがプラグイン・レジストリを列挙できます。`diagnose.ts` を作成します:

```ts
import { FiberState, type Context } from '@deepseek-ai/cordis'

export const name = 'diagnose'

export function apply(ctx: Context) {
  setTimeout(() => {
    for (const runtime of ctx.registry.values()) {
      for (const fiber of runtime.fibers) {
        if (fiber.state === FiberState.PENDING) {
          console.log(`${fiber.name} is PENDING — a required service is missing`)
        }
      }
    }
  }, 500)
}
```

そして、満たせない依存関係を持つプラグイン `needs-timer.ts` です:

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'needs-timer'
export const inject = ['timer']

export function apply(ctx: Context) {
  console.log('needs-timer loaded')
}
```

```yaml
- name: './needs-timer.ts'
- name: './diagnose.ts'
```

実行します（通常の `node --import tsx ../../vendor/cordis/bin.js`。Ctrl-C で停止）:

```
needs-timer is PENDING — a required service is missing
```

`inject: ['timer']` にはプロバイダーがありません。リストに `- name: '@deepseek-ai/cordis-plugin-timer'` を追加すると、プラグインがロードされます。プラグインが何もせず何も報告しない場合は、その fiber の状態を検査してください。PENDING フィルターなしで反復すると、ローダー自身のプラグイン（Loader、Include）も ACTIVE な fiber として表示されます。プラグインが設定ファイル自体をマウントするからです。

次: [harness へ](07-into-the-harness.md) — 実際の harness サービスに対する同じパターン。

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
