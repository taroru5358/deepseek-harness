# スコープ付き登録

日本語 | [English](../../../docs/subsystems/scope.md) | [中文](../../../docs/subsystems/scope.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/scope.md](../../../docs/subsystems/scope.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[scope パッケージ](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/core/scope)は、1 つの登録コンテキストが agent ごとの可視性と共有ライフタイム所有権の両方を意味するようにする、アイデンティティ、キャリア、スコープ付きレイヤーの語彙を提供する。これは Cordis サービスではなくライブラリ・プリミティブである。ライフサイクルの根拠は [agent-scope ランタイム設計リファレンス](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md)が所有し、呼び出し可能な API とフィルタリングのセマンティクスはパッケージの [README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/scope/README.md)が所有する。

ソース: [`packages/core/scope/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/scope/src/index.ts) と [`packages/core/scope/src/store.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/scope/src/store.ts)。

## アイデンティティとディスパッチ・キャリア

`ScopeKey` は不透明なオブジェクト・アイデンティティである。同梱のループはライブの `Agent` オブジェクトを自身のキーとして使うが、プリミティブはそのオブジェクトを決して検査しない。

```ts type-equiv
/** An opaque, identity-compared scope key. */
type ScopeKey = object
```

`Scoped<T>` は、`scopeTarget(base, key)` が返す不透明なルーティング・レシーバーに付くコンパイル時のブランドである。スコープでフィルターされたイベント宣言はこのキャリアを `this` 型として要求し、実際のイベント主体は明示的な引数のままである。

```ts type-equiv
/**
 * A routing-only event receiver built by {@link scopeTarget}. The type
 * parameter records the subject type for dispatch checking; the carrier does
 * not expose the subject's properties. Event payloads carry the real subject.
 */
type Scoped<T extends object> = object & { readonly [ScopedBrand]: T }
```

## 所有される登録コンテキスト

`Scope` はタグ付きの登録コンテキストと 2 つの解体パスを対にする。`rawDispose` は、順序付けられた複合エフェクトが必要とする正確な Cordis disposer のアイデンティティを保つ。`dispose()` は、直接の呼び出し元と競合する呼び出し元のための公開された共有の静止境界である。

```ts type-equiv
/** A minted registration scope and its quiescent disposal boundaries. */
interface Scope {
  /** Context through which scope-owned registrations are made. */
  ctx: Context
  /** Exact Cordis disposer, used when nesting this scope in an ordered composite effect. */
  rawDispose: () => Promise<void> | void
  /** Dispose every scope-owned registration; racing calls await the same completion. */
  dispose(): Promise<void>
}
```

## スコープ付きレジストリ・レイヤー

`ScopeLayer` は、グローバルまたは正確なスコープのレベルでの、1 つのレジストリの完全な寄与を表す。具体的なレイヤーは複数の名前付きおよび匿名のテーブルを集約してよい。レイヤー全体の空性により、`ScopedLayers` は兄弟テーブルを破棄せずにスコープ付き状態を回収できる。

```ts type-equiv
/** One scope's aggregate contribution to a registry. */
interface ScopeLayer {
  /** Whether every table in this layer is empty. */
  isEmpty(): boolean
}
```

`ScopedLayers<L>` は、eager なグローバル・レイヤーと遅延作成される正確なスコープのレイヤーを所有する。読み取りはレイヤーを作成しない。`peek(undefined)` はオーバーレイがないことを意味し、`merge()` は挿入順のグローバルな名前付きエントリーに続いてスコープ付きのシャドウを実体化する。登録は可視性と Cordis エフェクト所有権の両方に 1 つのコンテキストを使い、任意の通知の前に 1 つの同期 undo を集め、Cordis の正確な disposer を返し、スコープ付きレイヤーをその完全な `ScopeLayer` が空のときにのみ回収する。

`NamedEntries<V>` は挿入順のルックアップと、呼び出し元が所有する重複エラーを伴うライブ・イテレーションを提供する。`AnonymousEntries<V>` はすべての追加に一意のアイデンティティを与え、等しい値が独立したままになるようにする。イテレーションは 1 つの空でないテーブル世代内でライブに留まり、テーブルを排出すると既存のイテレーターが後の挿入から切り離される。どちらも冪等な正確なエントリーの undo を返す。共有の `EntryValues` 実装インターフェースは public ではない。
