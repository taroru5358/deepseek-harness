# コンピューター操作

日本語 | [English](../../../docs/subsystems/computer-use.md) | [中文](../../../docs/subsystems/computer-use.zh.md)

> これは非公式の日本語訳です。英語版（[docs/subsystems/computer-use.md](../../../docs/subsystems/computer-use.md)）が正です。翻訳時点: 2026-10-31、upstream commit `ce7073d51254c26cd63b22a60835b8f948deb447`。

コンピューター操作により、モデルは設定されたプロバイダーを通じてローカル・デスクトップを観察し操作できる。DSH の共有ケイパビリティは **computer use** と呼ばれ、**Cua Driver** は上流の実装を指す。

[Computer Use](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/cua-bundle/README.md)は、macOS の CUA 権限セットアップのための任意の実験的スイッチを提供する。

[`macOS CUA プロバイダー`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/cua-macos/README.md)は[共有 CUA オブジェクトとエグゼキューター](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/cua/README.md)を使い、これらは User Questions とプロファイル設定を通じてアプリケーションの同意を強制する。

CUA プロバイダーは、システム権限の状態、明示的なシステム要求、アプリケーションの取り消しのために `ctx.cuaAccess` を所有する。その[権限ページ](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/cua/README.md)がこれらの制御を投影し、アプリ／ウィンドウ操作はディスパッチ前にプロバイダーの同意ポリシーを強制する。

| 権限の種類 | 意味 |
|---|---|
| `CuaApplicationId`、`CuaApplicationGrant` | プラットフォームが提供するアプリケーション・アイデンティティと、表示される保存済みの判断。 |
| `CuaSystemPermission` | 人間が選択するアクセシビリティまたは画面収録。 |
| `CuaPermissionProcess`、`CuaSystemPermissionState` | ネイティブの権限チェック、観測されたヘルパー／親プロセス、ヘルパー起動時に macOS のプロセス責任が識別したアプリケーション。任意のシステム問い合わせやアプリケーション検索が利用できない場合、アプリケーションのメタデータは存在しない。 |
| `CuaApplicationState` | 保存済みの許可、拒否されたアプリケーション、永続化が利用可能かどうか。セッション限りの許可はプロバイダー内部に留まる。システムの観測とアプリケーション・アイコンは別々に問い合わせられるため、ページは各結果を届いた順に表示できる。 |

## プロバイダーの選択

[`dsh-computer-use`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/computer-use/computer-use/README.md)と 1 つのプロバイダーを同じコンポジションにマウントする。両方の Cua Driver プロバイダーは実験的な公開 npm パッケージであり、明示的な有効化を必要とする。

| プロバイダー | ランタイム |
|---|---|
| [Cua Driver MCP](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/computer-use-cua-driver-mcp/README.md) | MCP 経由で接続された、すでにインストール済みの `cua-driver` 実行ファイル |
| [Cua Driver native](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/computer-use-cua-driver-native/README.md) | npm 依存関係とともにインストールされるプラットフォーム・ネイティブのランタイム |

各プロバイダーは自身の上流ツール・カタログを提供する。共有サービスは名前だけを登録し、2 つ目のプロバイダーを拒否する。同じ名前の別インスタンスも含まれる。共通のデスクトップ操作メソッドや、モデルが制御するセレクターは持たない。

## 寿命とデスクトップの共有

プロバイダーはツールと所有リソースをシャットダウンしている間、登録を保持する。起動の失敗は試行された登録を解放する。MCP プロバイダーは再接続の間も登録を保持する。

登録された 1 つのプロバイダーが、Session のためにデスクトップを予約するわけではない。呼び出し元は Session と別々の DSH プロセスをまたいで、観察・操作・検証の完全なワークフローを調整する。キャンセルされた呼び出しは、デスクトップがすでに受け取った入力を取り消せない。

## 結果とプラットフォーム要件

ツールは通常の実行パイプラインと Session ログを使う。添付ストアを持つ画像対応のモデル・ルートは永続的なスクリーンショットを受け取り、画像非対応のルートは既存の MCP 画像診断を受け取る。インストール、権限、プラットフォームの制限はプロバイダーの README が所有する。

[決定記録](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-12-computer-use-provider-registration.md)は、登録のみのサービスと 2 つの Cua Driver 統合について説明する。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

`scripts/gen-cordis-catalog.ts` によってソースから生成される（doc-sync の `pnpm run verify-cordis-catalog` で最新性を検証し、`pnpm run gen-cordis-catalog` で再生成する）。言語側はロケール固有のペア文書パスだけが異なる。シグネチャブロックは `ts cordis-catalog` フェンスを使い、元のソース JSDoc を保持する。ディスパッチモードは [primer](../cordis-primer.md#dispatch-modes) で定義され、フレームワーク継承の `ctx` API は [cordis-api/inherited.md](../../../docs/cordis-api/inherited.md) にある。

<a id="ctxcomputeruse--computeruseregistry"></a>

### `ctx.computerUse` — `ComputerUseRegistry`

共有 computer-use サービス内の、任意の 1 つのプロバイダー登録を所有する。

```ts cordis-catalog
/**
 * Reserve the sole provider slot until the contribution is disposed.
 * A second registration fails even when it repeats the current name. Providers
 * must stop their tools and await owned work before releasing this registration.
 * @param name - provider-owned name used in registration diagnostics.
 * @returns the effect disposer for this exact registration.
 */
register(name: ComputerUseProviderName): () => Promise<void>
```

ソース: [`packages/computer-use/computer-use/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/computer-use/computer-use/src/index.ts)

<a id="ctxcuaaccess--cuaaccess"></a>

### `ctx.cuaAccess` — `CuaAccess`

プロバイダーが所有するネイティブ権限とアプリケーション判断のためのブラウザ制御。

```ts cordis-catalog
/** Read PTC execution state separately from native permission checks.
 * @returns Whether the shared runtime retains memory and has an editable profile entry.
 */
@Remote ptcConfiguration(): { persistent: boolean; writable: boolean }

/** Persist in-memory PTC execution through the existing profile editor and Loader lifecycle.
 * @returns Completion after the shared runtime reload; already-persistent runtimes are left intact.
 */
@Remote async enablePersistentPtc(): Promise<void>

/**
 * Read fresh stored application decisions without issuing a native request.
 * @returns Saved grants, refused applications and durable-storage availability.
 */
@Remote applications(): Promise<CuaApplicationState>

/**
 * Inspect system privileges independently of application decisions and icons.
 * @returns Fresh native preflight and launch metadata.
 */
@Remote async systemStatus(): Promise<CuaSystemPermissionState>

/**
 * Load presentation icons separately from application decisions.
 * @returns Optional native PNG icons keyed by executable path; applications without paths are skipped.
 */
@Remote async applicationIcons(): Promise<Record<string, string>>

/**
 * Request a system privilege explicitly selected by the user.
 * @param permission macOS privilege to request.
 * @returns Preflight state after requesting; a false value never implies consent.
 */
@Remote async requestPermission(permission: CuaSystemPermission): Promise<CuaSystemPermissionState>

/**
 * Open the macOS privacy pane selected by the user.
 * @param permission Corresponding system privilege.
 * @returns Completion after macOS accepted the open request.
 */
@Remote async openSettings(permission: CuaSystemPermission): Promise<void>

/**
 * Remove saved and session access and join affected operations.
 * @param id Application identity displayed on the permission page.
 * @returns Completion after persistence and cancellation; dispatched input cannot be undone.
 */
@Remote revoke(id: CuaApplicationId): Promise<void>

/**
 * Clear this application's refusals across sessions so each session may ask for access again.
 * @param id Refused or revoked application displayed on the permission page.
 * @returns Completion after clearing refusals; no permission is granted and no action is replayed.
 */
@Remote async retryApproval(id: CuaApplicationId): Promise<void>
```

ソース: [`packages/experimental/cua/src/access.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/experimental/cua/src/access.ts)
<!-- END GENERATED cordis-surface -->
