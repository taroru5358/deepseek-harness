# 5. 設定

日本語 | [English](../../../docs/cordis-tutorial/05-config.md) | [中文](../../../docs/cordis-tutorial/05-config.zh.md)

> これは非公式の日本語訳です。英語版（[docs/cordis-tutorial/05-config.md](../../../docs/cordis-tutorial/05-config.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

各 `cordis.yml` エントリーは `config` ブロックを持て、プラグインは `apply` が実行される前にそれを検証するスキーマを宣言します。不正な設定は正確なエラーでロードを失敗させます。プラグインが中途半端に設定された状態で開始することはありません。

## 設定可能なプラグイン

`tmp/cordis-tutorial` に `config-demo.ts` を作成します:

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'

export const name = 'config-demo'

export interface Config {
  greeting: string
  targets: string[]
}

export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
  targets: Schema.array(String).default(['world']),
})

export function apply(ctx: Context, config: Config) {
  for (const target of config.targets) {
    console.log(`${config.greeting}, ${target}!`)
  }
}
```

export される `Config` は、同じ名前の TypeScript インターフェースと実行時スキーマの両方です。コンシューマーは型を得て、Cordis はバリデータを得ます。このリポジトリはスキーマに [Schemastery](https://github.com/shigma/schemastery) を使います。Cordis 自体は任意の [Standard Schema](https://standardschema.dev/) バリデータを受け付けるため、`Config` として export された単なるオブジェクトは機能しません。

設定します:

```yaml
- name: './config-demo.ts'
  config:
    targets: ['alpha', 'beta']
```

実行します:

```
Hello, alpha!
Hello, beta!
```

`greeting` は省略されたため、スキーマの既定値がそれを埋めました。`apply` は常に完全で検証済みの設定を受け取ります。

## 失敗を明示する

次に、不正なものを与えます:

```yaml
- name: './config-demo.ts'
  config:
    targets: 'not-an-array'
```

```
ValidationError: invalid config:
  - $.targets expected array but got not-an-array (at targets)
```

プラグインの fiber は FAILED になり、このチュートリアルのランチャーはエラーを出力した後にステータス 1 で終了します。プラグインは、利用できないリソースやプロバイダーを名指しするスキーマ上有効な設定も、その参照を解決できるようになった時点で拒否すべきです。

<a id="volatile-fields"></a>
## Volatile フィールド

各操作中にプラグインが読むフィールドには `.volatile()` を使います。そのフィールドへの変更は、プラグインを再マウントせずに安定した参照を更新します。`.get()` で読みます:

```ts
import type { Context } from '@deepseek-ai/cordis'
import type {} from '@deepseek-ai/cordis-plugin-loader'
import Schema from '@deepseek-ai/schemastery'

export const Config = Schema.object({
  greeting: Schema.string().default('Hello').volatile(),
})

export function apply(ctx: Context, config: ReturnType<typeof Config>) {
  ctx.on('loader/volatile-update', () => {
    console.log(config.greeting.get())
  })
}
```

`Config(raw)` の直接呼び出しも参照を返します。既定値のない任意フィールドにも参照があり、存在しない場合 `.get()` は `undefined` を返します。参照を保持するか、1回の操作のためにその値を取得します（`await` をまたぐ場合も含む）。その値を後の操作のために保持してはいけません。オブジェクトと配列の値は、プレーン・データのデタッチされ再帰的に凍結されたスナップショットです。関数、クラス・インスタンス、循環は拒否されます。

Loader はスキーマ宣言された volatile フィールドを無視して生の設定を比較します。volatile のみの変更はプラグインの `internal/config` フックを通じて解析され、検証されます。通常の実効値がすべて一致したままなら、Loader は新しい値を実行中の参照にコミットし、変更されたパスをキーの配列として持つインスタンス・ローカルの `loader/volatile-update` を1回 emit し、ターゲット・プラグインの `internal/update` をバイパスします。等しい値は通知しません。無効な候補はログに記録され、次のアクティベーションまで実行中の参照を変更しません。式の結果の変更を含む通常の実効値の変更と、通常フィールドの同時変更は、既存の更新フローを保ち、古い参照を変更せず通知もしません。Group/Include は引き続き子の更新をディスパッチします。直接の `fiber.update()`、コード置換、依存置換はそのライフサイクルを保ち、`noSave` は引き続き直接更新の永続化を制御します。通知はリソース再設定を待たずに通常の `emit` を使います。

volatile フィールドは固定のオブジェクト・パスで宣言するか、オブジェクトや配列全体を volatile にします。配列、辞書、union や intersect の分岐、lazy スキーマ、transform、別の volatile 値の中にある独立した参照は拒否されます。設定ファイルは通常の値を保存し、`simplify()` は参照を展開します。シリアライズされたスキーマは、スキーマ駆動フォームのためにフィールド型、既定値、ロール、volatile マーカーを保持します。既存の設定コンシューマーは独自の設定インターフェースを使い続けます。

## 計算された設定値

このリポジトリで使うローダーは、ロード時に計算する必要がある設定値のために `!!js` タグをサポートします:

```yaml
- name: './config-demo.ts'
  config:
    greeting: !!js process.env.DEMO_GREETING ?? 'Hello'
```

`!!js` は `config` 内と、エントリーの `disabled` フィールドでのみ機能します。`disabled: !!js ...` はマウントの決定ごとにローダー・コンテキストに対して評価されるため（このリポジトリの拡張）、行はプラットフォームや環境で自身をゲートできます。その他のメタデータ（`name`、`id`、`inject`、...）は静的で、式は通常の真値データになります。[loader 設定](../cordis-primer.md#loader-configuration)を参照してください。

次: [構成と HMR](06-composition-and-hmr.md) — `cordis.yml` をアプリケーションとして扱う。

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
