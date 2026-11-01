# 権限プリセット

日本語 | [English](../../../docs/subsystems/permission-presets.md) | [中文](../../../docs/subsystems/permission-presets.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/permission-presets.md](../../../docs/subsystems/permission-presets.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

[dsh-permission-presets](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/interaction/permission-presets)（`ctx.permissionPresets`、`PermissionPresetService`）の権限プリセット層は、2 つの独立した強制ノブ — [sandbox モード](sandbox.md)（`sandbox/mode`）と[承認ポリシー](approval.md)（`approval/policy`）— を名前付きプリセットに束ね、クライアントが 1 つの Permissions セレクターとして提供する。設定されたテーブルは将来のセッションの既定値を所有し、固定の `registerAuto(admit)` フックにより、[Auto レビュー](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/auto-review/README.md)統合が現行セッション限定のオプションを 1 つの effect ライフタイムの間公開できる。この層は任意であり、実行ポリシーを所有しない。プロンプトのナレーションとリプレイはそれぞれのノブ fold を読み続け、Auto レビューが追加の強制を所有する。[パッケージ README](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/interaction/permission-presets/README.md)が合成の状態と制限を所有し、[過去の sandbox 切り替え設計](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/archived/feature/2026-07-06-sandbox.md)が元のノブの根拠を記録する。

ソース: [`packages/interaction/permission-presets/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/interaction/permission-presets/src/index.ts)

## プリセット・テーブル

プリセットは 1 つの安定したキーを sandbox/承認のバンドルと任意のクライアント・プレゼンテーションにマップする。既定の設定テーブルは `workspace-write`（`workspace-write` + `ask`）と `danger-full-access`（`danger-full-access` + `never`）を出荷する。`custom` と `auto` は予約済みで設定できない。

```ts type-equiv
/** One preset's sandbox/approval bundle and optional client presentation. */
interface PresetSpec {
  /** The `sandbox/mode` value the preset writes through. */
  sandbox: SandboxMode
  /** The `approval/policy` value the preset writes through. */
  approval: ApprovalPolicy
  /** The display label a client shows for this preset; the raw table key when omitted. */
  name?: string
  /** One user-facing sentence on what the preset means; omitted when not configured. */
  description?: string
}
```

```ts type-equiv
/** The {@link PermissionPresetService} config: preset table and composition default. */
interface Config {
  /**
   * The preset table: name → knob bundle. Defaults to `workspace-write`
   * (workspace-write + ask) and `danger-full-access` (danger-full-access +
   * never). The names `custom` and `auto` are reserved for derived state and
   * the Auto review integration respectively.
   */
  presets: Record<string, PresetSpec>
  /**
   * Default for new sessions. When omitted, the preset matching the composed
   * sandbox and approval defaults is used.
   */
  defaultPreset: Volatile<string | undefined>
}
```

サービスは閉じ込めを行う `ctx.shell` エグゼキューターと `ctx.approval` を必要とし、設定の誤りはプラグイン・ロード時に失敗する。`custom` または `auto` という名前の設定エントリーは例外を投げ、閉じ込めを行わない（`sandboxMode` ケイパビリティ・ファクトがない）bash エグゼキューター上での合成も、プリセットが sandbox モードを束ねるため例外を投げる。

## 固定された現行セッションの Auto 登録

Auto 統合は、その effect ライフタイムの間 `registerAuto(admit)` を呼ぶ。このサービスは `auto` のアイデンティティとその `danger-full-access` プラス `ask` のバンドルを固定する。記録された Auto 選択は、委譲された子が固定する `never` ポリシーにも一致する。出荷されるクライアントのロケール辞書が Auto のラベルと説明を所有し、設定済みプリセットのプレゼンテーションは Host 所有のままである。呼び出し元は汎用のコントリビューション API を通じて別のプリセットを公開できない。Auto は設定済みプリセットの後に現れ、`permission.defaultPreset` 設定スキーマには決して入らず、effect が破棄されると消える。同期の `admit` コールバックは、Auto 選択が Session を変更する前、および保存された Auto Session が公開される前に実行されるので、統合が存在しない、または終了中でも永続アイデンティティが書き換えられることはない。

Auto の登録または削除は、ペイロードのない `permission-presets/catalog-changed` 通知を発行する。プロセスのコンシューマーは `catalog()` を呼ぶ前に購読し、各通知の後に選択可能なカタログ全体を読み直す。`permissions` Session projection は `currentValue` だけを含むので、カタログの変更は Session イベントを追記せず、Session projection フレームを公開せず、Session のシーケンスを変更しない。

## 現在のプリセットと派生 `custom`

`current(session)` は必須の `permissions` projection から有効なプリセットを導出する。このユニットはセッションの sandbox モード、承認ポリシー、記録された選択を fold する。その状態内で欠けている値は、エグゼキューターの設定モードと承認サービスの設定、次に `ask` にフォールバックする。projection キーが見つからない場合は明示的に失敗する。サービスは、まだ一致している選択（`never` 承認ポリシー下の記録済み Auto 選択を含む）を優先し、次に最初に一致する設定エントリーを選び、それ以外は `CUSTOM_PRESET`（`'custom'`）を返す。`custom` は派生専用である。クライアントは現在値として表示できるが、切り替え対象やイベント・ペイロードになることは決してない。

`names` は設定済みプリセットを宣言順で列挙し、統合がライブの間はその後に Auto を続ける。`catalog()` はそれらの選択可能なエントリーを 1 つのプロセス・レベルのスナップショットとして返す。`optionOf(name)` は利用可能なエントリー（ラベルはキーにフォールバックする）または派生 `custom` のプレゼンテーションを構築し、それ以外の名前では例外を投げる。クライアントはカタログを Session projection と結合する。`custom` は現在値にラベルを付けられるが、カタログ・エントリーになることは決してない。

```ts type-equiv
/** Presentation for an available preset or the derived `custom` current value. */
interface PresetOption {
  /** Stable option value: a configured preset key, live `auto`, or derived `custom`. */
  value: string
  /** The display label. */
  name: string
  /** One user-facing sentence on what the value means; omitted when not configured. */
  description?: string
}
```

## 切り替えと `permission/preset` イベント

`set(session, name)` はプリセットを解決し（未知の名前は例外を投げる）、該当する場合は Auto の受入を実行し、`name` が既に有効なプリセットでない限りログ専用の `permission/preset` イベントを追記し、その後、各ノブの有効値が変わるときだけ、それぞれのセッター — [dsh-sandbox-policy](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/sandbox/sandbox-policy) の `setSandboxMode` と [dsh-user-approval](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/interaction/user-approval) の `setApprovalPolicy` — を通じて書き込む。選択イベントは同じ turn のノブ・イベントに先行し、有効なプリセットを再選択しても何も追記されない。

`permission/preset` は永続的でログ専用のユーザー意図である。モデルのトランスクリプトには入らず（ノブ・イベントがコンシューマーを通じてモデル可視の結果を所有する）、2 つのプリセットが同じバンドルを共有するときに `current()` がユーザーの選んだプリセットを保持できるようにするために存在する。`permissions` projection はその選択を両方のノブ・イベントと fold し、復元された空のシードと新しいセッションを区別するために使う `session/end-seed` 境界を保持する。リプレイにはキャッチアップ状態や生ログの再スキャンは不要である。復元された `auto` 選択は、Agent の公開前にライブの Auto 登録を必要とする。完全なイベント宣言は[永続化ログのイベントカタログ](../../../docs/persistence-catalog.md) にあり、メソッド・シグネチャは生成された[サービス・カタログ](#ctxpermissionpresets--permissionpresetservice) にある。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。dispatch モードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxpermissionpresets--permissionpresetservice"></a>

### `ctx.permissionPresets` — `PermissionPresetService`

デプロイの設定済み権限プリセット、固定の Auto 統合フック、およびそれらの書き込み経路を所有する。閉じ込めを行う `ctx.shell` エグゼキューターと `ctx.approval` を必要とする。一致しないノブ値はエラーではなく CUSTOM_PRESET として報告される。

```ts cordis-catalog
/**
 * Read the complete process-level catalog exposed to current-session UI.
 * @returns every currently selectable preset in contribution order.
 */
@Remote('catalog') catalog(): PermissionCatalog

/**
 * Publish the fixed current-session Auto preset for the calling
 * integration's effect lifetime.
 * @param admit - synchronous gate run before live Auto selection or restore.
 * @returns the async effect disposer that removes Auto.
 */
registerAuto(admit: () => void): () => Promise<void>

/**
 * Resolve the preset matching the effective knob values. A still-matching
 * last selection wins shared-bundle ties, and a still-selected Auto also
 * matches the `never` approval policy; otherwise the first configured
 * match wins. Returns
 * {@link CUSTOM_PRESET} when no available preset matches.
 * @param session - the session whose knob state is read.
 * @returns the effective preset name, or `custom` when nothing matches.
 */
current(session: Session): string

/**
 * Resolve an available preset's knob bundle.
 * @param name - the preset name to resolve.
 * @returns the configured bundle.
 * @throws when `name` is neither configured nor the currently live Auto preset.
 */
resolve(name: string): PresetSpec

/**
 * Build the client option for an available preset or {@link CUSTOM_PRESET}.
 * A missing label falls back to the preset key.
 * @param name - a configured preset key, live `auto`, or `custom`.
 * @returns the option a client renders.
 * @throws when `name` is neither a configured preset, live `auto`, nor `custom`.
 */
optionOf(name: string): PresetOption

/**
 * Record a changed preset, then update each changed knob through its own
 * setter. Selecting the effective preset again appends nothing.
 * @param session - the session the switch belongs to.
 * @param name - the preset to switch to; unknown names throw.
 */
set(session: Session, name: string): void
```

型: [Session](session.md)

ソース: [`packages/interaction/permission-presets/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/interaction/permission-presets/src/index.ts)

<a id="permission-presets-events"></a>

### `permission-presets/*` イベント

<a id="permission-presetscatalog-changed--emit"></a>

#### `permission-presets/catalog-changed` — emit

選択可能なプロセス・カタログが変わった。設計上ペイロードを持たない。コンシューマーは先に購読し、その後カタログ全体を読み直す。

```ts cordis-catalog
/**
 * The selectable process catalog changed. Payload-free by design:
 * consumers subscribe first, then re-read the complete catalog.
 * @mode emit
 */
'permission-presets/catalog-changed'(): void
```

ソース: [`packages/interaction/permission-presets/src/types.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/interaction/permission-presets/src/types.ts)
<!-- END GENERATED cordis-surface -->
